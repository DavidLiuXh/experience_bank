* Controller这个角色是在kafka 0.8以后添加的,它负责的功能很多;
* Topic的创始, Partition leader的选取, Partition的增加, PartitionReassigned, PreferredReplicaElection, Topic的删除等;
***
# 选主
 Kafkak中有多处涉及到选主和failover, 比如Controller, 比如Partition leader. 我们先来看下和选主有关的类;
## LeaderElector
* 所在文件: core/src/main/scala/kafka/server/LeaderElector.scala
* 是个trait, 源码中的注释:
>This trait defines a leader elector If the existing leader is dead, this class will handle automatic re-election and if it succeeds, it invokes the leader state change callback

* 接口:
```
trait LeaderElector extends Logging {
       def startup // 启动
       def amILeader : Boolean //标识是否为主
       def elect: Boolean //选主
       def close  //关闭
}
```

## ZookeeperLeaderElector
* 所在文件: core/src/main/scala/kafka/server/ZookeeperLeaderElector.scala
* 实现了 trait LeaderElector
* 基于zookeeper临时节点的抢占式选主策略, 多个备选者都去zk上注册同一个临时节点, 但zk保证同时只有一个备选者注册成功, 此备选者即成为leader, 然后大家都watch这个临时节点, 一旦此临时节点消失, watcher被触发, 各备选者又一次开始抢占选主;
* `startup方法`: 先watch这个zk节点, 然后调用`elect`;
```
def startup {
    inLock(controllerContext.controllerLock) {
      controllerContext.zkUtils.zkClient.subscribeDataChanges(electionPath, leaderChangeListener)
      elect
    }
  }
```
* `elect方法`:


![zookeeper_leader_elect.png](http://upload-images.jianshu.io/upload_images/2020390-18667b87587a80c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* `controllerContext.zkUtils.zkClient.subscribeDataChanges(electionPath, leaderChangeListener)` 这个`leaderChangeListener`被触发时:
###### 1. 临时节点数据发生变化`handleDataChange`: 如果改变前是leader, 改变后不是leader, 则回调`onResigningAsLeader()`;
###### 2. 临时节点被删除`handleDataDeleted`: 如果当前是leader, 则回调`onResigningAsLeader()`并同次调用`elect`开始抢占式选主;

# KafkaController的选主与Failover
* 使用`ZookeeperLeaderElector`作选主和Failover
```
private val controllerElector = new ZookeeperLeaderElector(controllerContext, ZkUtils.ControllerPath, onControllerFailover,
    onControllerResignation, config.brokerId)
```
* 在zk上的临时节点: `ZkUtils.ControllerPath = /controller`
* `KafkaController::startup`:
```
def startup() = {
    inLock(controllerContext.controllerLock) {
      info("Controller starting up")
      registerSessionExpirationListener()
      isRunning = true
      controllerElector.startup
      info("Controller startup complete")
    }
  }
```
其中
`registerSessionExpirationListener()` 注册zk连接的状态回调,处理SessionExpiration;
`controllerElector.startup` 开始选主和Failover;
* `onControllerFailover`: 变为leader时被回调, 
设置当前broker的状态为`RunningAsController` 作下面的事情:
>   This callback is invoked by the zookeeper leader elector on electing the current broker as the new controller.
   It does the following things on the become-controller state change -
    1. Register controller epoch changed listener
    2. Increments the controller epoch
    3. Initializes the controller's context object that holds cache objects for current topics, live brokers and leaders for all existing partitions.
    4. Starts the controller's channel manager
    5. Starts the replica state machine
    6. Starts the partition state machine
