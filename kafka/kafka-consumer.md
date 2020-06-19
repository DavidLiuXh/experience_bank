* Kafka的消息消费是以消费的group为单位;
* 同属一个group中的多个consumer分别消费topic的不同partition;
* 同组内consumer的变化, partition变化,  coordinator的变化都会引发balance;
* 消费的offset的提交
* Kafka wiki: [Kafka Detailed Consumer Coordinator Design](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Detailed+Consumer+Coordinator+Design) 和 [Kafka Client-side Assignment Proposal](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Client-side+Assignment+Proposal)
---
###### GroupMetadata类
- 所在文件: `core/src/main/scala/kafka/coordinator/MemberMetadata.scala`
- 作用: 用来表示一个消费group的相关信息
- 当前group的状态: `private var state: GroupState = Stable`
   1. **Stable**: consumer group的balance已完成, 处于稳定状态;
   2. **PreparingRebalance**: 收到JoinRequest, consumer group需要重新作balance时的状态;
  3. **AwaitingSync**: 收到了所有需要的JoonRequest, 等待作为当前group的leader的consumer客户端提交balance的结果到`coordinator`;
  4. **Dead**: 当前的消费group不再有任何consumer成员时的状态;
- 当前group的成员相关信息:
   1. 成员信息: `private val members = new mutable.HashMap[String, MemberMetadata]`,
    每个成员都有一个memberId, 对应着`MemberMetadata`;
   2.  `var leaderId: String`: 对于group的balance, 简单来讲实际上是`Coordinator`收集了所有的consumer的信息后, 将其发送给group中的一个consumer, 这个consumer负责按一定的balance策略,将partition分配到不同的consumer, 这个分配结果会Sync回`Coordinator`, 然后再同步到各个consumer, 这个负责具体分配的consumer就是当前的`Leader`; 这个`Leader`的决定很简单, 谁第一个加入这个group的,谁就是leader;
   3. `var protocol: String`: 当前group组所采用的balance策略, 选取的规则是被当前所有member都支持的策略中最多的那一个;
   4. `var generationId`: 当前balance的一个标识id, 可以简单理解成是第几次作balance, 每次状态转换到`AwaitingSync`时, 其值就增加1;
###### GroupMetadataManager类
- 所在文件: `core/src/main/scala/kafka/coordinator/GroupMetadataManager.scala`
- 作用: 是比较核心的一个类, 负责所有group的管理, offset消息的读写和清理等, 下面我们一一道来
- **当前所有消费group的管理:**
  1. `private val groupsCache = new Pool[String, GroupMetadata]`: 缓存了所有`GroupMetadata`的信息;
  2. 针对`groupsCache`的管理接口:
```
def getGroup(groupId: String): GroupMetadata
def addGroup(group: GroupMetadata): GroupMetadata
def removeGroup(group: GroupMetadata)
```
- **__consumer_offsets topic的读写**
  1. 我们已经知道现在的kafka已经支持将offset信息保存到broker上, 实际上是保存到一个内部的topic上:`__consumer_offsets`, 写入其中的msg都包含有`key`
  2. `__consumer_offsets`这个topic里实际上保存两种类型消息:
       2.1  一部分是offset信息(`kafka.coordinator.OffsetsMessageFormatter`类型)的：
`[groupId,topic,partition]::[OffsetMetadata[offset,metadata],CommitTime ExprirationTime]`, 它的`key`是 `[groupId,topic,partition]`
     2.2 另一部分是group信息(`kafka.coordinator.GroupMetadataMessageFormatter`类型):
    `groupId::[groupId,Some(consumer),groupState,Map(memberId -> [memberId,clientId,clientHost,sessionTimeoutMs], ...->[]...)]`, 这部分实际上就是把当前`Stable`状态的`GroupMetadata`存到了`__consumer_offsets`里, , 它的`key`是 `groupId`
  3. offset和group信息的写入: 实际上是普通的消息写入没有本质上的区别, 可参考[Kafka是如何处理客户端发送的数据的？](http://www.jianshu.com/p/561dcaff9a0b), 这里的方法是`def store(delayedAppend: DelayedStore)`, 实现就是调用`replicaManager.appendMessages`来写入消息到log文件 
- **`__consumer_offsets` topic消息的加载** 
  1. `__consumer_offsets`作为一个topic, 也是有多个partiton的, 每个partiton也是有多个复本的, partition也会经历leader的选举,也会有故障转移操作;
    2. 当`__consumer_offsets`在某台broker上的partition成为`leader partition`时, 需要先从本地的log文件后加载offset,group相关信息到内存, 加载完成后才能对外提供读写和balance的操作;
   3. 具体实现: `def loadGroupsForPartition(offsetsPartition: Int,
                             onGroupLoaded: GroupMetadata => Unit)`
- **offset的相关操作**
   1. 使用者消费msg提交的offset, 不仅会写入到log文件后, 为了快速响应还会缓存在内存中, 对应`private val offsetsCache = new Pool[GroupTopicPartition, OffsetAndMetadata]`;
  2. 直接从内存中获取某一group对应某一topic的parition的offset信息:
`def getOffsets(group: String, topicPartitions: Seq[TopicAndPartition]): Map[TopicAndPartition, OffsetMetadataAndError]`
   3. 刷新offset: `offsetsCache`只保存最后一次提交的offset信息
`private def putOffset(key: GroupTopicPartition, offsetAndMetadata: OffsetAndMetadata)`
- **删除过期的offset消息**
  1. `GroupMetadataManager`在启动时会同时启动一个名为`delete-expired-consumer-offsets`定时任务来定时删除过期的offset信息;
   2. 从内存缓存中清除: `offsetsCache.remove(groupTopicAndPartition)`
   3. 从已经落地的log文件中清除: 实现就是向log里写一条payload为null的"墓碑"message作为标记, `__consumer_offsets`的清除策略默认是`compact`, 后面我们会单独开一章来讲日志的清除;
###### GroupCoordinator类
- 所在文件: `core/src/main/scala/kafka/coordinator/GroupCoordinator.scala`
- 核心类, 处理所有和消息消费相关的request:
 ```
        case RequestKeys.OffsetCommitKey => handleOffsetCommitRequest(request)
        case RequestKeys.OffsetFetchKey => handleOffsetFetchRequest(request)
        case RequestKeys.GroupCoordinatorKey => handleGroupCoordinatorRequest(request)
        case RequestKeys.JoinGroupKey => handleJoinGroupRequest(request)
        case RequestKeys.HeartbeatKey => handleHeartbeatRequest(request)
        case RequestKeys.LeaveGroupKey => handleLeaveGroupRequest(request)
        case RequestKeys.SyncGroupKey => handleSyncGroupRequest(request)
        case RequestKeys.DescribeGroupsKey => handleDescribeGroupRequest(request)
        case RequestKeys.ListGroupsKey => handleListGroupsRequest(request)
```
- 使用简单状态机来协调consumer group的balance;
- 下面我们假设在一个group:g1中启动两个consumer: c1和c2来消费同一个topic, 来看看状态机的转换
   1. 第一种情况: c1和c2分别启动:

![c2.jpg](http://upload-images.jianshu.io/upload_images/2020390-e6cefadde992fb70.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  2. 第二种情况: c1和c2已经在group中, 然后c1正常的退出离开

![c1.jpg](http://upload-images.jianshu.io/upload_images/2020390-1f0d1f28924cfaac.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
   3. 第二种情况: c1和c2已经在group中, 然后c1非正常退出,比如说进程被kill掉
       流程跟上面的2基本上一致, 只不过(1)这步的触发条件不是LeaveGroupRequest, 而是来自c1的heartbeat的onExpireHeartbeat;
   4. 第四种情况: c1和c2已经在group中, 然后这个topic的partition增加, 这个时候服务端是无法主动触发的,客户端会定时去服务端同步metadata信息, 从新的metadata信息中客户端会获知partition有了变化, 此时c1和c2会重新发送`JoinRequest`来触发新的balance;
   5. 还有其它的两种情况, 这里就不一一说明了,总之就是利用这个状态机的转换来作相应的处理.