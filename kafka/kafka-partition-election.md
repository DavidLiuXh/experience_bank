* 我们都知道, Kafka的每个Topic的存储在逻辑上分成若干个Partition,每个Partition又可以设置自己的副本Replica；
* 这样的设计就引出了几个概念:
   1. Partition: 消息在Kafka上存储的最小逻辑单元, 在物理上对应在不同的Broker机器上;
   2. Replica: 每个Partition可以设置自己的副本Partition, 这样主Partition叫作`Leader`, 副本叫作`Replica`；从灾备的角度考虑, 在物理上Replica尽量不要与Leader在同一台Broker物理机上;
   3. Ack: 客户端produce消息时, 可以设置Kafka服务端回应ack的策略:
       3.1 不用回Ack, 客户端发送效率最高, 但无法确认是否真的发送成功;
       3.2 仅Partition leader回ack, 发送效率次之, 可以确认Leader已经接收到消息;在这种情况下,如果leader挂了, 客户端将无法消费到这个消息;
       3.3 所有Replica(实际上这不是真的)都需要回ack, 发送效率最差, Replica需要从Leader拉取消息;
   4. ISR: `In Sync Replica`, 是所有Replica的一个子集. Partition的replica可能很多, 针对上面的3.3,如果需要所有replicat都拉取到消息后再回ack,发送效率会很差,因此Kafka用了折衷的办法, 仅需要ISR中的replica接收了消息即可.ISR中的replica的消息应一直与leader同步;
* 既然有`Leader`的角色,又有多个replica, 就存在一个在**选主**的问题, 我们就来讲下多种情况下的选主策略;
***
# PartitionLeaderSelector
* 所在文件: core/src/main/scala/kafka/controller/PartitionLeaderSelector.scala
* 这个`trait`, 各种选主策略类都实现了它.声明了如下的方法, 返回`LeaderAndIsr`类型的request
```
/**
   * @param topicAndPartition          The topic and partition whose leader needs to be elected
   * @param currentLeaderAndIsr        The current leader and isr of input partition read from zookeeper
   * @throws NoReplicaOnlineException If no replica in the assigned replicas list is alive
   * @return The leader and isr request, with the newly selected leader and isr, and the set of replicas to receive
   * the LeaderAndIsrRequest.
   */
  def selectLeader(topicAndPartition: TopicAndPartition, currentLeaderAndIsr: LeaderAndIsr): (LeaderAndIsr, Seq[Int])
```

# OfflinePartitionLeaderSelector
* 所在 core/src/main/scala/kafka/controller/PartitionLeaderSelector.scala
* 可用于Offline状态Partitions的选主,比如Topic刚刚创建后;
* 规则, 源码中的注释
> Select the new leader, new isr and receiving replicas (for the LeaderAndIsrRequest):
 1. If at least one broker from the isr is alive, it picks a broker from the live isr as the new leader and the live isr as the new isr.
 2. Else, if unclean leader election for the topic is disabled, it throws a NoReplicaOnlineException.
 3. Else, it picks some alive broker from the assigned replica list as the new leader and the new isr.
 4. If no broker in the assigned replica list is alive, it throws a NoReplicaOnlineException
 Replicas to receive LeaderAndIsr request = live assigned replicas
 5. Once the leader is successfully registered in zookeeper, it updates the allLeaders cache

* 翻译成图:

![PartitionLeaderSelector.png](http://upload-images.jianshu.io/upload_images/2020390-57909b2fe284a82c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# ReassignedPartitionLeaderSelector
* 所在 core/src/main/scala/kafka/controller/PartitionLeaderSelector.scala
* 用于Partitions再分配后的LeaderSelect;
* 规则:
>* New leader = a live in-sync reassigned replica
 * New isr = current isr
 * Replicas to receive LeaderAndIsr request = reassigned replicas

# ControlledShutdownLeaderSelector
* 所在 core/src/main/scala/kafka/controller/PartitionLeaderSelector.scala
* 用于ControllerShutdown时的leader select
* 规则:
> * New leader = replica in isr that's not being shutdown;
 * New isr = current isr - shutdown replica;
 * Replicas to receive LeaderAndIsr request = live assigned replicas

# NoOpLeaderSelector
* 所在 core/src/main/scala/kafka/controller/PartitionLeaderSelector.scala
* 其实什么都不作,返回当前的Leader, ISR和Replicas;
```
def selectLeader(topicAndPartition: TopicAndPartition, currentLeaderAndIsr: LeaderAndIsr): (LeaderAndIsr, Seq[Int]) = {
    warn("I should never have been asked to perform leader election, returning the current LeaderAndIsr and replica assignment.")
    (currentLeaderAndIsr, controllerContext.partitionReplicaAssignment(topicAndPartition))
  }
```