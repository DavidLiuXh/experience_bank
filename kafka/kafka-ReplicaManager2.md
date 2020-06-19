
* 发送到broker后，broker如何接收，如何存储;
---
#### KafkaApis中响应`LeaderAndIsr Request`
* 针对topic，KafkaController会进行partition选主，产生最新的isr, replicas, 然后发送到topic的各个replica，详细参考[Kafka集群建立过程分析](http://www.jianshu.com/p/fba24b16afd3)和[KafkaController分析5-Partition状态机](http://www.jianshu.com/p/2cae7ab344bf)
* `KafkaApis中`处理`LeaderAndIsr Request`并发送`LeaderAndIsrResponse`；
```
    val correlationId = request.header.correlationId
    val leaderAndIsrRequest = request.body.asInstanceOf[LeaderAndIsrRequest]

      def onLeadershipChange(updatedLeaders: Iterable[Partition], updatedFollowers: Iterable[Partition]) {
        // for each new leader or follower, call coordinator to handle consumer group migration.
        // this callback is invoked under the replica state change lock to ensure proper order of
        // leadership changes
        updatedLeaders.foreach { partition =>
          if (partition.topic == GroupCoordinator.GroupMetadataTopicName)
            coordinator.handleGroupImmigration(partition.partitionId)
        }
        updatedFollowers.foreach { partition =>
          if (partition.topic == GroupCoordinator.GroupMetadataTopicName)
            coordinator.handleGroupEmigration(partition.partitionId)
        }
      }

      val responseHeader = new ResponseHeader(correlationId)
      val leaderAndIsrResponse=
        if (authorize(request.session, ClusterAction, Resource.ClusterResource)) {
          val result = replicaManager.becomeLeaderOrFollower(correlationId, leaderAndIsrRequest, metadataCache, onLeadershipChange)
          new LeaderAndIsrResponse(result.errorCode, result.responseMap.mapValues(new JShort(_)).asJava)
        } else {
          val result = leaderAndIsrRequest.partitionStates.asScala.keys.map((_, new JShort(Errors.CLUSTER_AUTHORIZATION_FAILED.code))).toMap
          new LeaderAndIsrResponse(Errors.CLUSTER_AUTHORIZATION_FAILED.code, result.asJava)
        }

      requestChannel.sendResponse(new Response(request, new ResponseSend(request.connectionId, responseHeader, leaderAndIsrResponse)))
```
其中最主要的操作调用`ReplicaManager.becomeLeaderOrFollower`来初始化`Partition`
```
val result = replicaManager.becomeLeaderOrFollower(correlationId, leaderAndIsrRequest, metadataCache, onLeadershipChange)
```

* `ReplicaManager.becomeLeaderOrFollower`
 1. 判断`LeaderAndIsr`请求中的controllerEpoch和`ReplicaManager`保存的controllerEpoch(在处理`UpdateMetadata Request`时更新, 参见[Kafka集群Metadata管理](http://www.jianshu.com/p/b8e7996ea9af)), 如果本地存的controllerEpoch大，则忽略当前的`LeaderAndIsr`请求, 产生`BecomeLeaderOrFollowerResult(responseMap, ErrorMapping.StaleControllerEpochCode)`；
 2. 处理`leaderAndISRRequest.partitionStates`中的第个partition state;
  2.1 创建`Partition`对象，这个我们后面会讲到;
  ```
allPartitions.putIfNotExists((topic, partitionId), new Partition(topic, partitionId, time, this))
  ```
 2.2 如果partitionStateInfo中的leaderEpoch更新，则存储它在`val partitionState = new mutable.HashMap[Partition, LeaderAndIsrRequest.PartitionState]()
`中
```
if (partitionLeaderEpoch < stateInfo.leaderEpoch) {
          if(stateInfo.replicas.contains(config.brokerId))
                 partitionState.put(partition, stateInfo)
}
```
2.3 分离出转换成leader和follower的partitions;
```
 val partitionsTobeLeader = partitionState.filter { case (partition, stateInfo) =>
          stateInfo.leader == config.brokerId
        }
  val partitionsToBeFollower = (partitionState -- partitionsTobeLeader.keys)
```
2.4 处理转换成leader
```
makeLeaders(controllerId, controllerEpoch, partitionsTobeLeader, correlationId, responseMap)
```
实现上干两件事:
   *停止从leader来同步消息*: `replicaFetcherManager.removeFetcherForPartitions(partitionState.keySet.map(new TopicAndPartition(_))) 
`,参见[ReplicaManager源码解析1-消息同步线程管理](http://www.jianshu.com/p/db2582925c10)
*调用Partition的makeLeader方法*:`partition.makeLeader(controllerId, partitionStateInfo, correlationId)`来作leader的转换
2.5 处理转换成follower
```
makeFollowers(controllerId, controllerEpoch, partitionsToBeFollower, leaderAndISRRequest.correlationId, responseMap, metadataCache)
```
2.6 启动HighWaterMarkCheckPointThread, 具体后面章节会讲到,
```
if (!hwThreadInitialized) {
          startHighWaterMarksCheckPointThread()
          hwThreadInitialized = true
}
```
2.7 回调`KafkaApis.handleLeaderAndIsrRequest.onLeadershipChange`
* 针对`makeLeaders`和`makeFollowers`的分析我们等分析完`Parition`, `ReplicaFetcherManager`后一并分析.
* LeaderAndIsr 请求响应流程图:


![LeaderAndIsr 请求响应.png](http://upload-images.jianshu.io/upload_images/2020390-bd58850b21ec3712.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)