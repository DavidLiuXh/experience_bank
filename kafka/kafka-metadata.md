* 对于集群中的每一个broker都保存着相同的完整的整个集群的metadata信息;
* metadata信息里包括了每个topic的所有partition的信息: leader, leader_epoch, controller_epoch, isr, replicas等;
* Kafka客户端从任一broker都可以获取到需要的metadata信息;
---
###### Metadata的存储在哪里 --- MetadataCache组件
* 在每个Broker的`KafkaServer`对象中都会创建`MetadataCache`组件, 负责缓存所有的metadata信息;
```
val metadataCache: MetadataCache = new MetadataCache(config.brokerId)
```

* 所在文件: core/src/main/scala/kafka/server/MetadataCache.scala
* 所有的metadata信息存储在map里, key是topic, value又是一个map, 其中key是parition id, value是`PartitionStateInfo`
```
private val cache: mutable.Map[String, mutable.Map[Int, PartitionStateInfo]] =
    new mutable.HashMap[String, mutable.Map[Int, PartitionStateInfo]]()
```
* `PartitionStateInfo`: 包括`LeaderIsrAndControllerEpoch`和Replica数组; 下面的`readFrom`方法从接受到的buffer构造一个`PartitionStateInfo`对象:
```
def readFrom(buffer: ByteBuffer): PartitionStateInfo = {
    val controllerEpoch = buffer.getInt
    val leader = buffer.getInt
    val leaderEpoch = buffer.getInt
    val isrSize = buffer.getInt
    val isr = for(i <- 0 until isrSize) yield buffer.getInt
    val zkVersion = buffer.getInt
    val replicationFactor = buffer.getInt
    val replicas = for(i <- 0 until replicationFactor) yield buffer.getInt
    PartitionStateInfo(LeaderIsrAndControllerEpoch(LeaderAndIsr(leader, leaderEpoch, isr.toList, zkVersion), controllerEpoch),
                       replicas.toSet)
  }
```

* `MetadataCache`还保存着推送过来的有效的broker信息
```
private var aliveBrokers: Map[Int, Broker] = Map()
```

###### MetadataCache如何获取和更新metadata信息
* `KafkaApis`对象处理`UpdateMetadataRequest`
```
case RequestKeys.UpdateMetadataKey => handleUpdateMetadataRequest(request)
```
* `handleUpdateMetadataRequest`:
```
    val updateMetadataRequest = request.requestObj.asInstanceOf[UpdateMetadataRequest]
    
    authorizeClusterAction(request)

    replicaManager.maybeUpdateMetadataCache(updateMetadataRequest, metadataCache)

    val updateMetadataResponse = new UpdateMetadataResponse(updateMetadataRequest.correlationId)
    requestChannel.sendResponse(new Response(request, new RequestOrResponseSend(request.connectionId, updateMetadataResponse)))
```
可以看到是调用了ReplicaManager.maybeUpdateMetadataCache方法, 里面又会调用到`MetadataCache.updateCache`方法

* `MetadataCache.updateCache`:
```
      aliveBrokers = updateMetadataRequest.aliveBrokers.map(b => (b.id, b)).toMap
      updateMetadataRequest.partitionStateInfos.foreach { case(tp, info) =>
        if (info.leaderIsrAndControllerEpoch.leaderAndIsr.leader == LeaderAndIsr.LeaderDuringDelete) {
          removePartitionInfo(tp.topic, tp.partition)
        } else {
          addOrUpdatePartitionInfo(tp.topic, tp.partition, info)
        }
      }
```
**干三件事**
 1. 更新`aliveBrokers`;
 2. 如果某个topic的的parition的leader是无效的, 则`removePartitionInfo(tp.topic, tp.partition)`;
 3. 新增或更新某个topic的某个parition的信息, `addOrUpdatePartitionInfo(tp.topic, tp.partition, info)`: 将信息meta信息保存到`MetadataCache`的`cache`对象中;

###### Metadata信息从哪里来
* 这个问题实际上就是在问`UpdateMetaRequest`是*谁*在*什么时候*发送的;
* 来源肯定是`KafkaController`发送的;
*  broker变动, topic创建, partition增加等等时机都需要更新metadata;

###### 谁使用metadata信息
* 主要是客户端, 客户端从metadata中获取topic的partition信息, 知道leader是谁, 才可以发送和消费msg;
* `KafkaApis`对象处理`MetadataRequest`
```
case RequestKeys.MetadataKey => handleTopicMetadataRequest(request)
```

* `KafkaApis.handleTopicMetadataRequest`:
```
    val metadataRequest = request.requestObj.asInstanceOf[TopicMetadataRequest]

    //if topics is empty -> fetch all topics metadata but filter out the topic response that are not authorized
    val topics = if (metadataRequest.topics.isEmpty) {
      val topicResponses = metadataCache.getTopicMetadata(metadataRequest.topics.toSet, request.securityProtocol)
      topicResponses.map(_.topic).filter(topic => authorize(request.session, Describe, new Resource(Topic, topic))).toSet
    } else {
      metadataRequest.topics.toSet
    }

    //when topics is empty this will be a duplicate authorization check but given this should just be a cache lookup, it should not matter.
    var (authorizedTopics, unauthorizedTopics) = topics.partition(topic => authorize(request.session, Describe, new Resource(Topic, topic)))

    if (!authorizedTopics.isEmpty) {
      val topicResponses = metadataCache.getTopicMetadata(authorizedTopics, request.securityProtocol)
      if (config.autoCreateTopicsEnable && topicResponses.size != authorizedTopics.size) {
        val nonExistentTopics: Set[String] = topics -- topicResponses.map(_.topic).toSet
        authorizer.foreach {
          az => if (!az.authorize(request.session, Create, Resource.ClusterResource)) {
            authorizedTopics --= nonExistentTopics
            unauthorizedTopics ++= nonExistentTopics
          }
        }
      }
    }

    val unauthorizedTopicMetaData = unauthorizedTopics.map(topic => new TopicMetadata(topic, Seq.empty[PartitionMetadata], ErrorMapping.TopicAuthorizationCode))

    val topicMetadata = if (authorizedTopics.isEmpty) Seq.empty[TopicMetadata] else getTopicMetadata(authorizedTopics, request.securityProtocol)
    val brokers = metadataCache.getAliveBrokers
    val response = new TopicMetadataResponse(brokers.map(_.getBrokerEndPoint(request.securityProtocol)), topicMetadata  ++ unauthorizedTopicMetaData, metadataRequest.correlationId)
    requestChannel.sendResponse(new RequestChannel.Response(request, new RequestOrResponseSend(request.connectionId, response)))
```

看着代码不少, 实际上比较简单:
 1. 先确定需要获取哪些topic的metadata信息,  如果request里未指定topic, 则获取当前所有的topic的metadata信息;
 2. 有效性验证,将topic分为`authorizedTopics`和`unauthorizedTopics`;
 3. 获取authorizedTopics的metadata, 注意`getTopicMetadata`方法是关键所在, 它会先筛选出当前不存在的topic, 如果`auto.create.topics.enable=true`, 则调用`AdminUtils.createTopic`先创建此topic, 但此时其PartitionStateInfo为空, 不过也会作为Metadata Response的一部分返回给客户端.