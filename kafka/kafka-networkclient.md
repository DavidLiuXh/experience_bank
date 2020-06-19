* KafkaController的作用前面我们已经[简单介绍](http://www.jianshu.com/p/04f6bd37d2ef)过, 基于此KafkaController需要与其他的broker node通信,发送请求；
* ControllerChannelManager用来管理与其他所有的broker node的网络连接和请求发送等;
***
# ControllerChannelManager
* 所在文件: core/src/main/scala/kafka/controller/ControllerChannelManager.scala
* 创建到各个broker node的连接, 每个连接对应一个新的线程
```
controllerContext.liveBrokers.foreach(addNewBroker(_))
```
* 创建到单个broker node的连接
```
private def addNewBroker(broker: Broker) {
    ...
    val networkClient = {
      val selector = new Selector(
        ...
      )
      new NetworkClient(
        selector,
        new ManualMetadataUpdater(Seq(brokerNode).asJava),
        config.brokerId.toString,
        1,
        0,
        Selectable.USE_DEFAULT_BUFFER_SIZE,
        Selectable.USE_DEFAULT_BUFFER_SIZE,
        config.requestTimeoutMs,
        time
      )
    }
...
    val requestThread = new RequestSendThread(config.brokerId, controllerContext, messageQueue, networkClient,
      brokerNode, config, time, threadName)
    requestThread.setDaemon(false)
    brokerStateInfo.put(broker.id, new ControllerBrokerStateInfo(networkClient, brokerNode, messageQueue, requestThread))
  }
```
使用[NetworkClient](http://www.jianshu.com/p/af2c48ad854d)连接到broker node, 使用[selector](http://www.jianshu.com/p/8cbc7618abcb)处理网络IO;
* 发送线程`RequestSendThread`, 继承自`ShutdownableThread`, 需要发送的request会被add到`val queue: BlockingQueue[QueueItem]`中, 然后在`doWork`中被不断取出`val QueueItem(apiKey, apiVersion, request, callback) = queue.take()
`, 再通过`clientResponse = networkClient.blockingSendAndReceive(clientRequest, socketTimeoutMs)`被发送直到`clientResponse`返回
* 主要处理下面三种类型的请求:
```
val response = ApiKeys.forId(clientResponse.request.request.header.apiKey) match {
            case ApiKeys.LEADER_AND_ISR => new LeaderAndIsrResponse(clientResponse.responseBody)
            case ApiKeys.STOP_REPLICA => new StopReplicaResponse(clientResponse.responseBody)
            case ApiKeys.UPDATE_METADATA_KEY => new UpdateMetadataResponse(clientResponse.responseBody)
            case apiKey => throw new KafkaException(s"Unexpected apiKey received: $apiKey")
          }
```
* 如果设置了回调, 则
```
if (callback != null) {
            callback(response)
}
```

# ControllerBrokerRequestBatch
* 所在文件: core/src/main/scala/kafka/controller/ControllerChannelManager.scala
* 使用`ControllerChannelManager`的`sendRequest`方法来批量发送请求到broker node;
* 主要处理以下三种请求:
```
 val leaderAndIsrRequestMap = mutable.Map.empty[Int, mutable.Map[TopicPartition, PartitionStateInfo]]
  val stopReplicaRequestMap = mutable.Map.empty[Int, Seq[StopReplicaRequestInfo]]
  val updateMetadataRequestMap = mutable.Map.empty[Int, mutable.Map[TopicPartition, PartitionStateInfo]]
```