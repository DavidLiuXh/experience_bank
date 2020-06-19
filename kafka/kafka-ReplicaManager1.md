* 现在的Kafka增加了高可用的特性，即增加了复本的特性，同时必然会引入选主，同步等复杂性;
* ReplicaManager负责消息的写入，消费在多复本间同步, 节点成为主或从的转换等等相关的操作;
* 这篇我们先集中介绍下ReplicaManager里用到的各种基础类库
***
###### OffsetCheckPoint类
* 文件：/core/src/main/scala/kafka/server/OffsetCheckPoint.scala
* 在kafka的log dir目录下有一文件：replication-offset-checkpoint, 以Topic+Partition为key, 记录其高水位的值。那么何为高水位？简单说就是已经复制到所有replica的最后提交的offset, 即所有ISR中的logEndOffset的最小值与leader的目前的高水位值的之间的大者.
* replication-offset-checkpoint文件结构很简单：
第一行：版本号，当前是0
第二行：当前写入的Topic+Partition的记录个数
其他每行: topic 空格 partition 空格 offset
* OffsetCheckPoint类实现了对这个文件的读写
每次写入的时修会先写到 replication-offset-checkpoint.tmp 的临时文件，读入后再作rename操作;
* recovery-point-offset-checkpoint文件格式与replication-offset-checkpointg一样，同样使用OffsetCheckPoint类来读写.

######AbstractFetcherManager类
* 文件:/core/src/main/scala/kafka/server/AbstractFetcherManager.scala
* 是个基类, 用于管理当前broker上的所有从leader的数据同步;
* 主要成员变量:`private val fetcherThreadMap = new mutable.HashMap[BrokerAndFetcherId, AbstractFetcherThread]`, 实现的拉取消息由`AbstractFetcherThread`来负责, 每个brokerId+fetcherId对应一个`AbstractFetcherThread`;
* `addFetcherForPartitions(partitionAndOffsets: Map[TopicAndPartition, BrokerAndInitialOffset])`: 创建并开始消息同步线程;
 其中最主要的操作是调用`AbstractFetcherThread`的`addPartitions`方法来告诉同步线程具体需要同步哪些partition;
* `def removeFetcherForPartitions(partitions: Set[TopicAndPartition]) `: 移出对某些partition的同步;
* `def shutdownIdleFetcherThreads()`: 如果某些同步线程负责同步的partition数量为0,则停掉该线程;
* `def closeAllFetchers()`: 停掉所有的同步线程
* `def createFetcherThread(fetcherId: Int, sourceBroker: BrokerEndPoint): AbstractFetcherThread`: 抽象方法, 由子类实现, 用于创建具体的同步线程;

######ReplicaFetcherManager类
* 文件:/core/src/main/scala/kafka/server/ReplicaFetcherManager.scala
* 继承自AbstractFetcherManager类
* 仅实现了`def createFetcherThread(fetcherId: Int, sourceBroker: BrokerEndPoint): AbstractFetcherThread `: 创建了`ReplicaFetcherThread`

###### AbstractFetcherThread类
* 文件: /core/src/main/scala/kafka/server/AbstractFetcherThread.scala
* 本身是个抽象基类, 实现了从partition的leader来同步数据的具体操作;
* `def addPartitions(partitionAndOffsets: Map[TopicAndPartition, Long])`: 添加需要同步的partition信息, 包换topic, partition和初始开始同步的offset;
 如果提供的初始offset无效, 则通过`handleOffsetOutOfRange(topicAndPartition)`方法来获取有效的初始offset, 这个方法的说明参见下面`ReplicaFetcherThread类`的分析;
* `def delayPartitions(partitions: Iterable[TopicAndPartition], delay: Long)`: 延迟同步某些partition, 通过`DelayItem`来实现;
* `def removePartitions(topicAndPartitions: Set[TopicAndPartition])`: 移除某些partition的同步;
* 此线程的执行体:
```
override def doWork() {
     val fetchRequest = inLock(partitionMapLock) {
      val fetchRequest = buildFetchRequest(partitionMap)
      if (fetchRequest.isEmpty) {
        trace("There are no active partitions. Back off for %d ms before sending a fetch request".format(fetchBackOffMs))
        partitionMapCond.await(fetchBackOffMs, TimeUnit.MILLISECONDS)
      }
      fetchRequest
    }

    if (!fetchRequest.isEmpty)
      processFetchRequest(fetchRequest)
  }
```
基本上就是作三件事: 构造FetchRequest, 同步发送FetchRequest并接收FetchResponse, 处理FetchResponse, 这三件事的实现调用了下列方法:
```
def processPartitionData(topicAndPartition: TopicAndPartition, fetchOffset: Long, partitionData: PD)

  // handle a partition whose offset is out of range and return a new fetch offset
  def handleOffsetOutOfRange(topicAndPartition: TopicAndPartition): Long

  // deal with partitions with errors, potentially due to leadership changes
  def handlePartitionsWithErrors(partitions: Iterable[TopicAndPartition])

  protected def buildFetchRequest(partitionMap: Map[TopicAndPartition, PartitionFetchState]): REQ

  protected def fetch(fetchRequest: REQ): Map[TopicAndPartition, PD]
```
它们都是在具体的子类中实现, 我们在下面的 `ReplicaFetcherThread类` 中作说明.

###### ReplicaFetcherThread类
* 文件: /core/src/main/scala/kafka/server/ReplicaFetcherThread.scala
* `handleOffsetOutOfRange`:处理从leader同步数据时,当前replica的的初始offset为-1的情况
 1. `earliestOrLatestOffset(topicAndPartition, ListOffsetRequest.LATEST_TIMESTAMP,
      brokerConfig.brokerId)` 先通过向Leader发送`OffsetRequest`来获取leader当前的LogEndOffset;
 2. **如果Leader的LogEndOffset小于当前replica的logEndOffset**, 这原则上不可能啊,除非是出现了`Unclean leader election`:即ISR里的broker都挂了,然后ISR之外的一个replica作了主;
 3. 如果broker的配置不允许`Unclean leader election`, 则`Runtime.getRuntime.halt(1)
`;
 4. 如果broker的配置允许`Unclean leader election`, 则当前replica本地的log要作truncate, truncate到Leader的LogEndOffset;
 5.  **如果Leader的LogEndOffset大于当前replica的logEndOffset**, 说明Leader有有效的数据供当前的replica来同步,那么剩下的问题就是看从哪里开始同步了;
 6. `earliestOrLatestOffset(topicAndPartition, ListOffsetRequest.EARLIEST_TIMESTAMP,
        brokerConfig.brokerId)` 通过向Leader发送`OffsetRequest`来获取leader当前有效的最旧Offset: StartOffset;
 7. 作一次truncate, 从startOffset开始追加:`replicaMgr.logManager.truncateFullyAndStartAt(topicAndPartition, leaderStartOffset)`
* `def buildFetchRequest(partitionMap: Map[TopicAndPartition, PartitionFetchState]): FetchRequest`: 构造FetchRequest
```
  val requestMap = mutable.Map.empty[TopicPartition, JFetchRequest.PartitionData]

    partitionMap.foreach { case ((TopicAndPartition(topic, partition), partitionFetchState)) =>
      if (partitionFetchState.isActive)
        requestMap(new TopicPartition(topic, partition)) = new JFetchRequest.PartitionData(partitionFetchState.offset, fetchSize)
    }

    new FetchRequest(new JFetchRequest(replicaId, maxWait, minBytes, requestMap.asJava))
```
这个没什么好说的,就是按照FetchRequest的协议来;
* `def fetch(fetchRequest: REQ): Map[TopicAndPartition, PD]`: 发送FetchRequest请求,同步等待FetchResponse的返回
```
    val clientResponse = sendRequest(ApiKeys.FETCH, Some(fetchRequestVersion), fetchRequest.underlying)
    new FetchResponse(clientResponse.responseBody).responseData.asScala.map { case (key, value) =>
      TopicAndPartition(key.topic, key.partition) -> new PartitionData(value)
    }
```
使用`NetworkClient`来实现到leader broker的连接,请求的发送和接收,
使用`kafka.utils.NetworkClientBlockingOps._`实现了这个网络操作的同步阻塞方式.
这个实现可参见[KafkaController分析2-NetworkClient分析](http://www.jianshu.com/p/af2c48ad854d)
* `def processPartitionData(topicAndPartition: TopicAndPartition, fetchOffset: Long, partitionData: PartitionData)`: 处理拉取过来的消息
```
  try {
      val TopicAndPartition(topic, partitionId) = topicAndPartition
      val replica = replicaMgr.getReplica(topic, partitionId).get
      val messageSet = partitionData.toByteBufferMessageSet
      warnIfMessageOversized(messageSet)

      if (fetchOffset != replica.logEndOffset.messageOffset)
        throw new RuntimeException("Offset mismatch: fetched offset = %d, log end offset = %d.".format(fetchOffset, replica.logEndOffset.messageOffset))
      replica.log.get.append(messageSet, assignOffsets = false)
      val followerHighWatermark = replica.logEndOffset.messageOffset.min(partitionData.highWatermark)
      replica.highWatermark = new LogOffsetMetadata(followerHighWatermark)
      trace("Follower %d set replica high watermark for partition [%s,%d] to %s"
            .format(replica.brokerId, topic, partitionId, followerHighWatermark))
    } catch {
      case e: KafkaStorageException =>
        fatal("Disk error while replicating data.", e)
        Runtime.getRuntime.halt(1)
    }
```
干三件事:
 1. 消息写入以相应的replica;
 2. 更新replica的highWatermark
 3. 如果有`KafkaStorageException`异常,就退出啦~~
* ` def handlePartitionsWithErrors(partitions: Iterable[TopicAndPartition])`: 对于在同步过程中发生错误的partition,会调用此方法处理:
```
delayPartitions(partitions, brokerConfig.replicaFetchBackoffMs.toLong)
```
目前的作法是将此partition的同步操作延迟一段时间.