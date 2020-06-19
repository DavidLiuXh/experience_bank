* 首先我们知道客户端如果想发送数据，必须要有topic, topic的创建流程可以参考[Kafka集群建立过程分析](http://www.jianshu.com/p/fba24b16afd3)
* 有了topic, 客户端的数据实际上是发送到这个topic的partition, 而partition又有复本的概念，关于partition选主和复本的产生可参考[KafkaController分析4-Partition选主](KafkaController分析4-Partition选主)和[ReplicaManager源码解析2-LeaderAndIsr 请求响应](http://www.jianshu.com/p/41786e564d42)
* 关于Partition的从复本是如何从主拉取数据的，可以参考[ReplicaManager源码解析1-消息同步线程管理](http://www.jianshu.com/p/db2582925c10)
---
* 客户端的ProduceRequest如何被Kafka服务端接收?又是如何处理? 消息是如何同步到复本节点的? 本篇文章都会讲到, 实际上就是综合运用了上面第三点中的内容
* 上一节我们讲到所有的Request最终都会进入到`KafkaApis::handle`方法后根据`requestId`作分流处理, `ProduceRequest`也不例外;
***

# Topic的Leader和Follower角色的创建
- 之前在[ReplicaManager源码解析2-LeaderAndIsr 请求响应](http://www.jianshu.com/p/41786e564d42)中留了个尾巴，现在补上;
- 通过[Kafka集群建立过程分析](http://www.jianshu.com/p/fba24b16afd3)我们知道，Kafkaf集群的Controller角色会监听zk上`/brokers/topics`节点的变化，当有新的topic信息被写入后，Controller开始处理新topic的创建工作;
- Controller 使用[Partition状态机](http://www.jianshu.com/p/2cae7ab344bf)和[Replica状态机](http://www.jianshu.com/p/719c809a674a)来选出新topic的各个partiton的主，isr列表等信息;
- Controller 将新topic的元信息通知给集群中所有的broker, 更新每台borker的Metadata cache;
- Controller 将新topic的每个partiton的leader, isr , replica list信息通过[LeaderAndIsr Request](http://www.jianshu.com/p/41786e564d42)发送到对应的broker上;
- `ReplicaManager::becomeLeaderOrFollower` 最终会处理Leader或Follower角色的创建或转换;
- Leader角色的创建或转换：
   1. 停掉partition对应的复本同步线程; `replicaFetcherManager.removeFetcherForPartitions(partitionState.keySet.map(new TopicAndPartition(_)))`
  2. 将相应的partition转换成Leader
 `partition.makeLeader(controllerId, partitionStateInfo, correlationId)`,其中最重要的是`leaderReplica.convertHWToLocalOffsetMetadata()`, 在Leader replica上生成新的`high watermark`;
- Follower角色的创建或转换:
  1. 将相应的partition转换成Follower
`partition.makeFollower(controllerId, partitionStateInfo, correlationId)`
  2. 停掉已存在的复本同步线程
`replicaFetcherManager.removeFetcherForPartitions(partitionsToMakeFollower.map(new TopicAndPartition(_)))`
  3. 截断Log到当前Replica的high watermark
`logManager.truncateTo(partitionsToMakeFollower.map(partition => (new TopicAndPartition(partition), partition.getOrCreateReplica().highWatermark.messageOffset)).toMap)`
  4. 重新开启当前有效复本的同步线程
`replicaFetcherManager.addFetcherForPartitions(partitionsToMakeFollowerWithLeaderAndOffset)`, 同步线程会不停发送`FetchRequest`到Leader来拉取新的消息
# 客户端消息的写入
- kafka客户端的ProduceRequest只能发送给Topic的某一partition的Leader
- ProduceRequest在Leader broker上的处理 KafkaApis::handleProducerRequest
   1. 使用`authorizer`先判断是否有未验证的`RequestInfo`
`
val (authorizedRequestInfo, unauthorizedRequestInfo) =  produceRequest.data.partition  {
      case (topicAndPartition, _) => authorize(request.session, Write, new Resource(Topic, topicAndPartition.topic))
    }
`
   2.  如果`RequestInfo`都是未验证的,则不会处理请求中的数据
`sendResponseCallback(Map.empty)`
  3. 否则, 调用`replicaManager`来处理消息的写入;
  4. 流程图:
![handlerequest.png](http://upload-images.jianshu.io/upload_images/2020390-651d659c2206b2c0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- Leader通过调用`ReplicaManager::appendMessages`，将消息写入本地log文件(虽写入了log文件，但只是更新了LogEndOffset, 还并未更新HighWaterMark, 因此consumer此时无法消费到)，同时根据客户端所使用的ack策略来等待写入复本;
  1. 等待复本同步的反馈，利用了延迟任务的方式,其具体实现可参考[DelayedOperationPurgatory--谜之炼狱](http://www.jianshu.com/p/bbb1c4f45b4e), 
`val delayedProduce = new DelayedProduce(timeout, produceMetadata, this, responseCallback)
`
  2. 尝试是否可以立即完成上面1中的延迟任务，如果不行才将其加入到 **delayedProducePurgatory**中，
`delayedProducePurgatory.tryCompleteElseWatch(delayedProduce, producerRequestKeys)`
  3. 当这个Partition在本地的isr中的replica的LEO都更新到大于等于Leader的LOE时，leader的HighWaterMark会被更新，此地对应的`delayedProduce`完成，对发送消息的客户端回response, 表明消息写入成功(这个下一小节后细说);
   4. 如果在`delayedProduce`没有正常完成前，其超时了，对发送消息的客户端回response, 表明消息写入失败;
- Partition在本地的isr中的replica的LEO如何更新呢？
   1. 前面说过Follower在成为Follower的同时会开启`ReplicaFetcherThread`，通过向Leader发送`FetchRequest`请求来不断地从Leader来拉取同步最新数据, `ReplicaManager::fetchMessage`处理`FetchRequest`请求，从本地log文件中读取需要同步的数据，然后更新本地对应的`Replica`的LogEndOffset, 同时如果所有isr中的最小的LogEndOffset都已经大于当前Leader的HighWaterMark了, 那么Leader的HighWaterMark就可以更新了, 同时调用`ReplicaManager::tryCompleteDelayedProduce(new TopicPartitionOperationKey(topicAndPartition))`来完成对客户端发送消息的回应.
   2. 从上面的1中我们看到实际上发送`FetchRequest`的replica还未收到Response,这个`Leader`的HighWaterMark可能已经就更新了;
- 对于Replica的FetchRequest的回应
  1. 在`ReplicaManager::fetchMessage`, 调用`readFromLocalLog`从本地log中读取消息后,先判断是否可以立即发送`FetchRequest`的response: 
`if(timeout <= 0 || fetchInfo.size <= 0 || bytesReadable >= fetchMinBytes || errorReadingData)`

   > // respond immediately if 
     //                      1) fetch request does not want to wait
    //                        2) fetch request does not require any data
    //                        3) has enough data to respond
    //                        4) some error happens while reading data

   2. 如查不能立即发送, 需要构造`DelayedFetch`来延迟发送`FetchRequest`的response,
       这可能是`FetchRequset`中所请求的Offset, FileSize在当前的Leader上还不能满足,需要等待; 当`Replica::appendToLocaLog`来处理`ProduceRequest`请求是会尝试完成此`DelayedFetch`;