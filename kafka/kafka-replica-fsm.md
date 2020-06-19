# Replica的状态
Replica有7种状态:
* NewReplica: 在partition reassignment期间KafkaController创建New replica;
* OnlineReplica: 当一个replica变为一个parition的assingned replicas时, 其状态变为OnlineReplica, 即一个有效的OnlineReplica. Online状态的parition才能转变为leader或isr中的一员;
* OfflineReplica: 当一个broker down时, 上面的replica也随之die, 其状态转变为Onffline;
* ReplicaDeletionStarted: 当一个replica的删除操作开始时,其状态转变为ReplicaDeletionStarted;
* ReplicaDeletionSuccessful: Replica成功删除后,其状态转变为ReplicaDeletionSuccessful;
* ReplicaDeletionIneligible: Replica成功失败后,其状态转变为ReplicaDeletionIneligible;
* NonExistentReplica:  Replica成功删除后, 从ReplicaDeletionSuccessful状态转变为NonExistentReplica状态.
* 状态转换图:

![ReplicaStateMachine.png](http://upload-images.jianshu.io/upload_images/2020390-882acb4a68e5ce01.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# ReplicaStateMachine
* 所在文件: core/src/main/scala/kafka/controller/ReplicaStateMachine.scala
* `startup`: 启动`ReplicaStateMachine`
 1. `initializeReplicaState`: 初始化每个replica的状态, 如果replica所在的broker是live状态,则此replica的状态为`OnlineReplica`
```
for((topicPartition, assignedReplicas) <- controllerContext.partitionReplicaAssignment) {
      val topic = topicPartition.topic
      val partition = topicPartition.partition
      assignedReplicas.foreach { replicaId =>
        val partitionAndReplica = PartitionAndReplica(topic, partition, replicaId)
        controllerContext.liveBrokerIds.contains(replicaId) match {
          case true => replicaState.put(partitionAndReplica, OnlineReplica)
          case false =>
            replicaState.put(partitionAndReplica, ReplicaDeletionIneligible)
        }
      }
    }
```
 2. 处理可以转换到`Online`状态的Replica, `handleStateChanges(controllerContext.allLiveReplicas(), OnlineReplica)`, 并且发送`LeaderAndIsrRequest`到各broker nodes: `handleStateChanges(controllerContext.allLiveReplicas(), OnlineReplica)`
```
case OnlineReplica =>
          assertValidPreviousStates(partitionAndReplica,
            List(NewReplica, OnlineReplica, OfflineReplica, ReplicaDeletionIneligible), targetState)
          replicaState(partitionAndReplica) match {
            case NewReplica =>
              // add this replica to the assigned replicas list for its partition
              val currentAssignedReplicas = controllerContext.partitionReplicaAssignment(topicAndPartition)
              if(!currentAssignedReplicas.contains(replicaId))
                controllerContext.partitionReplicaAssignment.put(topicAndPartition, currentAssignedReplicas :+ replicaId)
            case _ =>
              // check if the leader for this partition ever existed
              controllerContext.partitionLeadershipInfo.get(topicAndPartition) match {
                case Some(leaderIsrAndControllerEpoch) =>
                  brokerRequestBatch.addLeaderAndIsrRequestForBrokers(List(replicaId), topic, partition, leaderIsrAndControllerEpoch,
                    replicaAssignment)
                  replicaState.put(partitionAndReplica, OnlineReplica)
                case None => // that means the partition was never in OnlinePartition state, this means the broker never
              }
          }
          replicaState.put(partitionAndReplica, OnlineReplica)
```

* 监听broker改变
```
 private def registerBrokerChangeListener() = {
    zkUtils.zkClient.subscribeChildChanges(ZkUtils.BrokerIdsPath, brokerChangeListener)
  }
```

* 处理borker的改变事件`BrokerChangeListener() `:
针对broker的上下线,分别回调`controller.onBrokerStartup`或`controller.onBrokerFailure`
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
# 补一张图

![610532728.jpg](http://upload-images.jianshu.io/upload_images/2020390-c1421d2d5e75caa7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)