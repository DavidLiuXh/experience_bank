* 从本章开始我们来介绍一个kafka集群逐步建立的过程;
* 集群中只有一台broker;
* topic的创建;
* 增加多台broker;
* 扩展已存在topic的partition;
---
# 第一个broker(我们叫它B1)启动
* broker启动流程,请参考[Kafka初始化流程与请求处理](http://www.jianshu.com/p/eccb310c64c6);
* broker在启动过程中, 会先启动`KafkaController`, 因为此时只有一台broker B1, 它将被选为当前kafka集群的Controller,  过程可参考[KafkaController分析1-选主和Failover](http://www.jianshu.com/p/04f6bd37d2ef);
* 当broker B1变为Controller后会作一系列的初始化, 具体参考 [KafkaController分析7-启动流程](http://www.jianshu.com/p/a9f559ee6035), 包括以下几点:
>  a. 更新zk上的controller epoch信息;
    b. 注册zk上的broker/topic节点变化事件通知;
    c. 初始化ControllerContext, 主要是从zk上获取broker, topic, parition, isr, partition leader, replicas等信息;
    d. 启动ReplicaStateMachine;
    e. 启动PartitionStateMachine;
    f. 发送所有的partition信息(leader, isr, replica, epoch等)到所有的 live brokers;
    g. 如果允许自动leader rebalance的话, 则启动AutoRebalanceScheduler;

* Controller初始化完成后, `KafkaHealthcheck`启动,在zk的`/brokers`下面注册自己的信息,类似下面这样:
`{"jmx_port":-1,"timestamp":"1477624160337","endpoints":["PLAINTEXT://10.123.81.11:9092"],"host":"10.123.81.11","version":3,"port":9092}`
* `KafkaController`中的`ReplicaStateMachine`已经启动且注册了`BrokerChangeListener`事件通知, 因为当`KafkaHealthcheck`启动结束后,`BrokerChangeListener`被触发:
```
def handleChildChange(parentPath : String, currentBrokerList : java.util.List[String]) {
      inLock(controllerContext.controllerLock) {
        if (hasStarted.get) {
          ControllerStats.leaderElectionTimer.time {
            try {
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
            } catch {
              case e: Throwable => error("Error while handling broker changes", e)
            }
          }
        }
      }
    }
  }
```
**干三件事**:
 1. 更新`ControllerContext.liveBrokers`;
 2. 获取新增的broker列表, 回调`KafkaController.onBrokerStartup(newBrokerIds.toSeq);`
 3. 获取死掉的broker列表, 回调`KafkaController.onBrokerFailure(deadBrokerIds.toSeq)`;

* `KafkaController.onBrokerStartup`: 针对新增的brokers作处理, 由于现在只有一个broker并且也没有任何的topic, 因此这里基本上是什么都不会作;

# 创建Topic
* 目前kafka支持两种方式创建topic:
   1. 如果kafka启动时允许自动创建topic(可以在配置文件中指定`auto.create.topics.enable=true`), 则发送消息到kafka时,若topic不存在,会自动创建;
   2. 使用admin工具(bin/kafka-topics.sh)先行创建, 我们这里讲解这种方式;
*   在使用`bin/kafka-topic.sh`脚本来创建topic时, topic的config会被写入zk的/config/topics/[topic]下, topic的parition分配信息会被写入zk的/brokers/topics/[topic]下.
    其中parition的分配信息用户可以指定,也可由kafka-topic.sh脚本自动产生,产生规则如下:
   如查未指定开始位置,就随机选择一位置开始，通过轮询方式分配每个分区的第一个replica的位置, 然后每个partition剩余的replicas的位置紧跟着其第一个replica的位置.
    假设一个集群有5个broker, 有个topic有10个parition, 每个parition有3个复本,则分配如下图:


![1553745402.jpg](http://upload-images.jianshu.io/upload_images/2020390-dc6dfe1e6e01df5b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* `KafkaController`中的`PartitionStateMachine`组件在启动时注册了`TopicChangeListener`(监控听/brokers/topics),  此时被触发:
```
           val currentChildren = {
              import JavaConversions._
              (children: Buffer[String]).toSet
            }
            val newTopics = currentChildren -- controllerContext.allTopics
            val deletedTopics = controllerContext.allTopics -- currentChildren
            controllerContext.allTopics = currentChildren

            val addedPartitionReplicaAssignment = zkUtils.getReplicaAssignmentForTopics(newTopics.toSeq)
            controllerContext.partitionReplicaAssignment = controllerContext.partitionReplicaAssignment.filter(p =>
              !deletedTopics.contains(p._1.topic))
            controllerContext.partitionReplicaAssignment.++=(addedPartitionReplicaAssignment)
            if(newTopics.size > 0)
              controller.onNewTopicCreation(newTopics, addedPartitionReplicaAssignment.keySet.toSet)
```
**干三件事**
 1.  更新`ControllerContext.allTopics`;
 2. 更新`ControllerContext.partitionReplicaAssignment`;
 3. 回调`KafkaController.onNewTopicCreation`;

* `KafkaController.onNewTopicCreation`: 处理新topic的创建
```
    partitionStateMachine.registerPartitionChangeListener(topic)
    onNewPartitionCreation(newPartitions)
```
`onNewPartitionCreation`的处理逻辑:
```
    partitionStateMachine.handleStateChanges(newPartitions, NewPartition)
    replicaStateMachine.handleStateChanges(controllerContext.replicasForPartition(newPartitions), NewReplica)
    partitionStateMachine.handleStateChanges(newPartitions, OnlinePartition, offlinePartitionSelector)
    replicaStateMachine.handleStateChanges(controllerContext.replicasForPartition(newPartitions), OnlineReplica)
```
 
 1. 针对每个新topic注册`PartitionChangeListener`, 监控其partition数量的改变;
 2. `partitionStateMachine.handleStateChanges(newPartitions, NewPartition)`: 将partition状态变为`NewPartition`;
 3. `replicaStateMachine.handleStateChanges(controllerContext.replicasForPartition(newPartitions), NewReplica)`:将replica状态变为`NewReplica`, 由于目前partition并没有进行选主操作,因此无其他操作被触发;
 4. `partitionStateMachine.handleStateChanges(newPartitions, OnlinePartition, offlinePartitionSelector)`: 
   4.1 将partition状态由`NewPartition` -> `OnlinePartition`;
   4.2 选取replicas列表的head作为leader, 将leader, isr信息写入zk的`/brokers/topics/[topic]/partitions/[partition id]/state`
   4.3 `BrokerRequestBatch.addLeaderAndIsrRequestForBrokers`: 构造 LeaderAndIsr Request,发送到各live broker, 这个request由broker内部的`ReplicaManager`组件处理,我们后面会有专门的章节来分析它;
  4.4 `replicaStateMachine.handleStateChanges(controllerContext.replicasForPartition(newPartitions), OnlineReplica)
`: 将replica状态由`NewReplica` -> `OnlineReplica`;

# 第二个broker(我们叫它B2)启动
* `KafkaController`组件依然启动, 选主时发现已有controller存在则不继续进行选主,但仍监听`LeaderChangeListener`事件;
* `KafkaHealthcheck`启动,在zk的`/brokers`下面注册自己的信息;
* 第一个broker B1此时作为Controller, `BrokerChangeListener`被触发, 获取新增的broker列表, 回调`KafkaController.onBrokerStartup(newBrokerIds.toSeq);`
 1. `sendUpdateMetadataRequest`: 发送所有topic的partitionStateInfos到各broker;
 2. 针对新启动的broker, 调用`replicaStateMachine.handleStateChanges`更新replica状态到`OnlineReplica`:
```
val allReplicasOnNewBrokers = controllerContext.replicasOnBrokers(newBrokersSet)
    replicaStateMachine.handleStateChanges(allReplicasOnNewBrokers, OnlineReplica)
```
 3. `partitionStateMachine.triggerOnlinePartitionStateChange()`：针对new and offline partitions进行选主;

# 针对已存在的topic, 在第二个broker B2上新增一个patition
* 通过`kafka-topics.sh`脚本的alter命令, 在zk的/brokers/topics/[topic]下更新新增的partition的replicas信息;
* `KafkaController`中的`PartitionStateMachine`组件监听的`AddPartitionsListener`事件被触发:
```
          val partitionReplicaAssignment = zkUtils.getReplicaAssignmentForTopics(List(topic))
          val partitionsToBeAdded = partitionReplicaAssignment.filter(p =>
            !controllerContext.partitionReplicaAssignment.contains(p._1))
          if(controller.deleteTopicManager.isTopicQueuedUpForDeletion(topic))
            error("Skipping adding partitions %s for topic %s since it is currently being deleted"
                  .format(partitionsToBeAdded.map(_._1.partition).mkString(","), topic))
          else {
            if (partitionsToBeAdded.size > 0) {
              info("New partitions to be added %s".format(partitionsToBeAdded))
              controllerContext.partitionReplicaAssignment.++=(partitionsToBeAdded)
              controller.onNewPartitionCreation(partitionsToBeAdded.keySet.toSet)
            }
          }
```
**干三件事**
 1. 获取topic新增的partition的replicas信息;
 2. 更新`ControllerContext.partitionReplicaAssignment`;
 3. 回调`KafkaController.onNewPartitionCreation`;
* 处理partition的新增:
```
def onNewPartitionCreation(newPartitions: Set[TopicAndPartition]) {
    info("New partition creation callback for %s".format(newPartitions.mkString(",")))
    partitionStateMachine.handleStateChanges(newPartitions, NewPartition)
    replicaStateMachine.handleStateChanges(controllerContext.replicasForPartition(newPartitions), NewReplica)
    partitionStateMachine.handleStateChanges(newPartitions, OnlinePartition, offlinePartitionSelector)
    replicaStateMachine.handleStateChanges(controllerContext.replicasForPartition(newPartitions), OnlineReplica)
  }
```