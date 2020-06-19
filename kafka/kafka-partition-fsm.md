# Partition状态
Partition有如下四种状态
* NonExistentPartition: 这个partition还没有被创建或者是创建后又被删除了;
* NewPartition: 这个parition已创建, replicas也已分配好,但leader/isr还未就绪;
* OnlinePartition: 这个partition的leader选好;
* OfflinePartition: 这个partition的leader挂了,这个parition状态为OfflinePartition;
* 状态转换图:

![PartitionStateTransaction.png](http://upload-images.jianshu.io/upload_images/2020390-a5e5e8ea84a903f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# PartitionStateMachine
* 所在文件: core/src/main/scala/kafka/controller/PartitionStateMachine.scala
* `startup`: 启动PartitionStateMachine
 1. `initializePartitionState()`: 初始化已经存在的Partition的状态
```
for((topicPartition, replicaAssignment) <- controllerContext.partitionReplicaAssignment) {
      // check if leader and isr path exists for partition. If not, then it is in NEW state
      controllerContext.partitionLeadershipInfo.get(topicPartition) match {
        case Some(currentLeaderIsrAndEpoch) =>
          // else, check if the leader for partition is alive. If yes, it is in Online state, else it is in Offline state
          controllerContext.liveBrokerIds.contains(currentLeaderIsrAndEpoch.leaderAndIsr.leader) match {
            case true => // leader is alive
              partitionState.put(topicPartition, OnlinePartition)
            case false =>
              partitionState.put(topicPartition, OfflinePartition)
          }
        case None =>
          partitionState.put(topicPartition, NewPartition)
      }
    }
```
 2. `triggerOnlinePartitionStateChange`: 更新当前所有parititon的状态, 其中包括partition 选主, IRS的分配等操作, 将产生的`LeaderAndIsrRequest`, `UpdateMetadataRequest`通过[ControllerBrokerRequestBatch](http://www.jianshu.com/p/a06c81f09ec8) 发送到各个broker node;
```
      brokerRequestBatch.newBatch()

      for((topicAndPartition, partitionState) <- partitionState
          if(!controller.deleteTopicManager.isTopicQueuedUpForDeletion(topicAndPartition.topic))) {
        if(partitionState.equals(OfflinePartition) || partitionState.equals(NewPartition))
          handleStateChange(topicAndPartition.topic, topicAndPartition.partition, OnlinePartition, controller.offlinePartitionSelector,
                            (new CallbackBuilder).build)
      }
      brokerRequestBatch.sendRequestsToBrokers(controller.epoch)
```

  3. `handleStateChange`: partition状态转换处理, 代码看着有点多,实现上没什么特别的,就是之前介绍过的一些partition选主, isr分配, 会生成`LeaderAndIsrRequest`, `UpdateMetadataRequest`, 添加到[ControllerBrokerRequestBatch](http://www.jianshu.com/p/a06c81f09ec8)里,等待发送到各broker node:`brokerRequestBatch.sendRequestsToBrokers(controller.epoch)`
```
targetState match {
        case NewPartition =>
          // pre: partition did not exist before this
          assertValidPreviousStates(topicAndPartition, List(NonExistentPartition), NewPartition)
          partitionState.put(topicAndPartition, NewPartition)
          // post: partition has been assigned replicas
        case OnlinePartition =>
          assertValidPreviousStates(topicAndPartition, List(NewPartition, OnlinePartition, OfflinePartition), OnlinePartition)
          partitionState(topicAndPartition) match {
            case NewPartition =>
              // initialize leader and isr path for new partition
              initializeLeaderAndIsrForPartition(topicAndPartition) //初次分配leader
            case OfflinePartition =>
              electLeaderForPartition(topic, partition, leaderSelector) //使用[PartitionLeaderSelector](http://www.jianshu.com/p/505fa1f9b61a)选主
            case OnlinePartition => // invoked when the leader needs to be re-elected
              electLeaderForPartition(topic, partition, leaderSelector)
            case _ => // should never come here since illegal previous states are checked above
          }
          partitionState.put(topicAndPartition, OnlinePartition)
           // post: partition has a leader
        case OfflinePartition =>
          // pre: partition should be in New or Online state
          assertValidPreviousStates(topicAndPartition, List(NewPartition, OnlinePartition, OfflinePartition), OfflinePartition)
          // should be called when the leader for a partition is no longer alive
          partitionState.put(topicAndPartition, OfflinePartition)
          // post: partition has no alive leader
        case NonExistentPartition =>
          // pre: partition should be in Offline state
          assertValidPreviousStates(topicAndPartition, List(OfflinePartition), NonExistentPartition)
          partitionState.put(topicAndPartition, NonExistentPartition)
          // post: partition state is deleted from all brokers and zookeeper
      }
```

* `registerListeners`: PartitionStateMachine的另一重要作用就是监听zk上Topic的改变和删除,其实就是watch相关的zk节点,
   1. 监听zk节点: `/brokers/topics`, topic的增加, 回调处理`TopicChangeListener`
```
private def registerTopicChangeListener() = {
    zkUtils.zkClient.subscribeChildChanges(BrokerTopicsPath, topicChangeListener)
  }
```
   2. 监听zk节点: `/admin/delete_topics`, topic的删除, 回调处理`DeleteChangeListener`
```
private def registerDeleteTopicListener() = {
    zkUtils.zkClient.subscribeChildChanges(DeleteTopicsPath, deleteTopicsListener)
  }
```
 
   3. 监听zk节点: `/brokers/topics/[topic]`, topic的partition的增加, 回调处理`AddPartitionsListener`
```
def registerPartitionChangeListener(topic: String) = {
    addPartitionsListener.put(topic, new AddPartitionsListener(topic))
    zkUtils.zkClient.subscribeDataChanges(getTopicPath(topic), addPartitionsListener(topic))
  }
   ```
# 补一张图

![2017500806.jpg](http://upload-images.jianshu.io/upload_images/2020390-faa9511012b34857.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)s