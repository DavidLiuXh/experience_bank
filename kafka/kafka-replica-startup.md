###### 前面我们已经分析了KafkaController中使用的一系列组件, 从本章开始,我们开始介绍KafkaController的各个功能:
###### [KafkaController分析1-选主和Failover](http://www.jianshu.com/p/04f6bd37d2ef)
###### [KafkaController分析2-NetworkClient分析](http://www.jianshu.com/p/af2c48ad854d)
###### [KafkaController分析3-ControllerChannelManager](http://www.jianshu.com/p/a06c81f09ec8)
###### [KafkaController分析4-Partition选主](http://www.jianshu.com/p/505fa1f9b61a)
###### [KafkaController分析5-Partition状态机](http://www.jianshu.com/p/2cae7ab344bf)
###### [KafkaController分析6-Replica状态机](http://www.jianshu.com/p/719c809a674a)

###### KafkaController启动流程
* 注册zk的SessionExpiration事件通知:`registerSessionExpirationListener`, 当session到期且新session建立后,进行controller的重新选主;
```
def handleNewSession() {
      info("ZK expired; shut down all controller components and try to re-elect")
      inLock(controllerContext.controllerLock) {
        onControllerResignation()
        controllerElector.elect
      }
    }
```

* 启动 ZookeeperLeaderElector:`controllerElector.startup`. 如果当前broker成功选为Controller, 则`onControllerFailover`回调被触发.
```
      readControllerEpochFromZookeeper()
      incrementControllerEpoch(zkUtils.zkClient)
      registerReassignedPartitionsListener()
      registerIsrChangeNotificationListener()
      registerPreferredReplicaElectionListener()
      partitionStateMachine.registerListeners()
      replicaStateMachine.registerListeners()
      initializeControllerContext()
      replicaStateMachine.startup()
      partitionStateMachine.startup()
      brokerState.newState(RunningAsController)
      maybeTriggerPartitionReassignment()
      maybeTriggerPreferredReplicaElection()
      sendUpdateMetadataRequest(controllerContext.liveOrShuttingDownBrokerIds.toSeq)
      if (config.autoLeaderRebalanceEnable) {
        autoRebalanceScheduler.startup()
        autoRebalanceScheduler.schedule("partition-rebalance-thread", checkAndTriggerPartitionRebalance,
          5, config.leaderImbalanceCheckIntervalSeconds.toLong, TimeUnit.SECONDS)
      }
      deleteTopicManager.start()
```

 1. 更新zk上的controller epoch信息;
 2. 注册zk上的broker/topic节点变化事件通知;
 3. 初始化ControllerContext, 主要是从zk上获取broker, topic, parition, isr, partition leader, replicas等信息;
 4. 启动[ReplicaStateMachine](http://www.jianshu.com/p/719c809a674a);
 5. 启动[PartitionStateMachine](http://www.jianshu.com/p/2cae7ab344bf);
 6. 发送所有的partition信息(leader, isr, replica, epoch等)到所有的 live brokers;
 7. 如果允许自动leader rebalance的话, 则启动AutoRebalanceScheduler;
 8. 启动TopicDeletionManager;

* KafkaController的启动图解:

![KafkaController.png](http://upload-images.jianshu.io/upload_images/2020390-30c1e64b1b7ad335.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)