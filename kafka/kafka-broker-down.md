* 在实际应用中broker可能因为机器，硬件，网络，进程自身等原因挂掉;
* 本章我们来看下一个broker挂掉后整个kafka集群会发生什么事情。
---
###### 挂掉的broker不是集群的Controller
* 在[Kafka集群建立过程分析](http://www.jianshu.com/p/fba24b16afd3)和[KafkaController分析6-Replica状态机](KafkaController分析6-Replica状态机)我们讲过，`KafkaController`组件中的`ReplicaStateMachine`对象在启动时会注册监听`BrokerChangeListener`事件;
* 当一个broker挂掉后，其在zk的`/brokers/ids`下面的节点信息将被自动删除;
* ReplicaStateMachine`的`BrokerChangeListener`将触发：
```
   val curBrokerIds = currentBrokerList.map(_.toInt).toSet
   val newBrokerIds = curBrokerIds -- controllerContext.liveOrShuttingDownBrokerIds
   val newBrokerInfo = newBrokerIds.map(zkUtils.getBrokerInfo(_))
   val newBrokers = newBrokerInfo.filter(_.isDefined).map(_.get)
   val deadBrokerIds = controllerContext.liveOrShuttingDownBrokerIds -- curBrokerIds
   controllerContext.liveBrokers = curBrokerIds.map(zkUtils.getBrokerInfo(_)).filter(_.isDefined).map(_.get)
newBrokers.foreach(controllerContext.controllerChannelManager.addBroker(_))
              deadBrokerIds.foreach(controllerContext.controllerChannelManager.removeBroker(_))
   if(newBrokerIds.size > 0)
          controller.onBrokerStartup(newBrokerIds.toSeq)
   if(deadBrokerIds.size > 0)
          controller.onBrokerFailure(deadBrokerIds.toSeq)
```
 1. 从zk返回了当前的broker列表信息;
 2. `val deadBrokerIds = controllerContext.liveOrShuttingDownBrokerIds -- curBrokerIds`获取到当前挂掉的broker ids;
 3. 更新`KafkaControllerContext.liveBrokers`;
 4. 回调`KafkaController.onBrokerFailure(deadBrokerIds.toSeq)`;

* Broker挂掉的逻辑处理：`KafkaController.onBrokerFailure(deadBrokerIds.toSeq)`
```
    val deadBrokersSet = deadBrokers.toSet
    val partitionsWithoutLeader = controllerContext.partitionLeadershipInfo.filter(partitionAndLeader =>
      deadBrokersSet.contains(partitionAndLeader._2.leaderAndIsr.leader) &&
    !deleteTopicManager.isTopicQueuedUpForDeletion(partitionAndLeader._1.topic)).keySet
    partitionStateMachine.handleStateChanges(partitionsWithoutLeader, OfflinePartition)
    partitionStateMachine.triggerOnlinePartitionStateChange()
    var allReplicasOnDeadBrokers = controllerContext.replicasOnBrokers(deadBrokersSet)
    val activeReplicasOnDeadBrokers = allReplicasOnDeadBrokers.filterNot(p => deleteTopicManager.isTopicQueuedUpForDeletion(p.topic))
  replicaStateMachine.handleStateChanges(activeReplicasOnDeadBrokers, OfflineReplica)
```
 1. 先处理`KafkacontrollerContext.partitionLeadershipInfo`(这里面保存着当前所有topic的各个partition的leader相关信息)，筛选出所有leader为当前挂掉的broker的`TopicAndPartiton`保存到`partitionsWithoutLeader`中；
 2. 将`partitionsWithoutLeader`中的partition状态转换成`OfflinePartition`;
 3. 通过`partitionStateMachine.triggerOnlinePartitionStateChange()`对于上面2中`OfflinePartition`状态的partition进行重新选主(`PartitonStateMachine.electLeaderForPartition`);
 4. 产生新的LeaderAndIsr Request发送到topic相关的replicas上;
 5. 产生新的UpdateMetadata Request发送到各broker上;
 6. `replicaStateMachine.handleStateChanges(activeReplicasOnDeadBrokers, OfflineReplica)
`: 将相应的replica状态转换为`OfflineReplica`;
 7. 在上面6中的状态转换时会调用`controller.removeReplicaFromIsr(topic, partition, replicaId)`, 生成新的LeaderAndIsr Request,  真正broker挂掉这种情况个人感觉这个调用是多余的,因为在上面的3中新的LeaderAndIsr Request已经发送;