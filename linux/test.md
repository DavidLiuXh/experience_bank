简书链接： Kafka运维填坑

* 前提: 只针对Kafka 0.9.0.1版本;
* 说是运维,其实偏重于问题解决;
* 大部分解决方案都是google而来, 我只是作了次搬运工;
* 有些问题的解决方案未必一定是通用的, 若应用到线上请慎重;
* 如有疏漏之处, 欢迎大家批评指正;
* 列表:
   1. **Replica无法从leader同步消息**
   2. **Broker到zk集群的连接不时会断开重断**
   3. **Broker重启耗时很久**
   4. **不允许脏主选举导致Broker被强制关闭**
   5. **Replica从错误的Partition leader上去同步数据**
   6. **__consumer_offsets日志无法被清除**
   7. **GC问题**
   8. **zk和kafka部署**
   9. **监控很重要**
   10. **大量异常: `Attempted to decrease connection count for address with no connections`**
  11. **新版sdk访问较旧版的kafka, 发送kafka不支持的request**
  12. **频繁FullGC**
---
###### Replica无法从leader同步消息
- 现象: 集群上某topic原来只有单复本, 增加双复本后,发现有些partition没有从leader同步数据,导致isr列表中一直没有新增的replica;
- 日志分析:
```
[2017-09-20 19:37:05,265] ERROR Found invalid messages during fetch for partition [xxxx,87] offset 1503297 error Message is corrupt (stored crc = 286782282, computed crc = 400317671) (kafka.server.ReplicaFetcherThread)
[2017-09-20 19:37:05,458] ERROR Found invalid messages during fetch for partition [xxxx,75] offset 1501373 error Message found with corrupt size (0) in shallow iterator (kafka.server.ReplicaFetcherThread)
[2017-09-20 19:37:07,455] ERROR [ReplicaFetcherThread-0-5], Error due to  (kafka.server.ReplicaFetcherThread)
kafka.common.KafkaException: error processing data for partition [xxxx,87] offset 1503346
        at kafka.server.AbstractFetcherThread$$anonfun$processFetchRequest$2$$anonfun$apply$mcV$sp$1$$anonfun$apply$2.apply(AbstractFetcherThread.scala:147)
        at kafka.server.AbstractFetcherThread$$anonfun$processFetchRequest$2$$anonfun$apply$mcV$sp$1$$anonfun$apply$2.apply(AbstractFetcherThread.scala:122)
        at scala.Option.foreach(Option.scala:257)
        at kafka.server.AbstractFetcherThread$$anonfun$processFetchRequest$2$$anonfun$apply$mcV$sp$1.apply(AbstractFetcherThread.scala:122)
        at kafka.server.AbstractFetcherThread$$anonfun$processFetchRequest$2$$anonfun$apply$mcV$sp$1.apply(AbstractFetcherThread.scala:120)
        at scala.collection.mutable.HashMap$$anonfun$foreach$1.apply(HashMap.scala:99)
        at scala.collection.mutable.HashMap$$anonfun$foreach$1.apply(HashMap.scala:99)
        at scala.collection.mutable.HashTable$class.foreachEntry(HashTable.scala:230)
        at scala.collection.mutable.HashMap.foreachEntry(HashMap.scala:40)
        at scala.collection.mutable.HashMap.foreach(HashMap.scala:99)
        at kafka.server.AbstractFetcherThread$$anonfun$processFetchRequest$2.apply$mcV$sp(AbractFeherThread.scala:120)
        at kafka.server.AbstractFetcherThread$$anonfun$processFetchRequest$2.apply(AbstractFetcherThread.scala:120)
        at kafka.server.AbstractFetcherThread$$anonfun$processFetchRequest$2.apply(AbstractFetcherThread.scala:120)
        at kafka.utils.CoreUtils$.inLock(CoreUtils.scala:262)
        at kafka.server.AbstractFetcherThread.processFetchRequest(AbstractFetcherThread.scala:118)
        at kafka.server.AbstractFetcherThread.doWork(AbstractFetcherThread.scala:93)
        at kafka.utils.ShutdownableThread.run(ShutdownableThread.scala:63)
Caused by: java.lang.RuntimeException: Offset mismatch: fetched offset = 1503346, log end offset = 1503297.
        at kafka.server.ReplicaFetcherThread.processPartitionData(ReplicaFetcherThread.scala:110)
        at kafka.server.ReplicaFetcherThread.processPartitionData(ReplicaFetcherThread.scala:42)
        at kafka.server.AbstractFetcherThread$$anonfun$processFetchRequest$2$$anonfun$apply$mcV$sp$1$$anonfun$apply$2.apply(AbstractFetcherThread.scala:138)
```
- 解决:
   1. Kafka 0.9.0.1版本的bug: [ReplicaFetcherThread stopped after ReplicaFetcherThread received a corrupted message](ReplicaFetcherThread stopped after ReplicaFetcherThread received a corrupted message)
  2. 升级版本 或者 按上面链接中Reporter给出的简单修复来避开这个问题;
- 深究:
   这个bug被触发实际是上下面这个导致:
` ERROR Found invalid messages during fetch for partition [qssnews_download,87] offset 1503297 error Message is corrupt (stored crc = 286782282, computed crc = 400317671) (kafka.server.ReplicaFetcherThread)`
当时触发这个bug的时, 恰逢相应的broker机器上硬盘出现了多个坏块, 但不能完全确定这个crc错误跟这个有关.这个也有个Kafka的issue: [Replication issues](Replication issues)
###### Broker到zk集群的连接不时会断开重断
- 现象: broker不时地和zk重新建立session;
- 日志分析: broker日志里报zk连接超时或不能从zk读取任何数据
- 解决: 增加broker的zk的session timeout时间, 不能完全解决,但会改善很多;
- 深究: 
   1.  目前用的kafka集群还是相对比较稳定, 但是这个zk超时问题真是百思不得其解啊.
broker在启动时会在zk上注册一个临时节点,表时自己已上线, 一旦session超时,此临时节点将被删除, 相当于此broker下线, 必然引起整个集群的抖动,可参考[KafkaController分析8-broker挂掉](KafkaController分析8-broker挂掉)
  2. zk为何会timeout, 根本原因未能准确定位,目前看到跟诸多因素有关,比如磁盘IO, CPU负载, GC等等吧;
###### Broker重启耗时很久
- 现象: broker重启下分耗时
- 日志分析: 重启时加载所有的log segments, rebuild index;
- 解决: 应该是stop时, 没有优雅的shutdown, 直接 kill -9导致;
- 深究: 
  1. 停止broker服务请使用kafka本身提供的脚本优雅shutdown；
  2. 在shutdown broker时确保相应的zk集群是可用状态, 否则可能无法优雅地shutdown broker.
###### 不允许脏主选举导致Broker被强制关闭
- 现象: 监控到集群中某台broker挂掉
- 日志分析: 
`[2016-02-25 00:29:39,236] FATAL [ReplicaFetcherThread-0-1], Halting because log truncation is not allowed for topic test, Current leader 1's latest offset 0 is less than replica 2's latest offset 151 (kafka.server.ReplicaFetcherThread)`
- 解决: 实际上是设置了`unclean.leader.election.enable=false`, 然后走到了代码里下面这段逻辑
```
  if (leaderEndOffset < replica.logEndOffset.messageOffset) {
      // Prior to truncating the follower's log, ensure that doing so is not disallowed by the configuration for unclean leader election.
      // This situation could only happen if the unclean election configuration for a topic changes while a replica is down. Otherwise,
      // we should never encounter this situation since a non-ISR leader cannot be elected if disallowed by the broker configuration.
      if (!LogConfig.fromProps(brokerConfig.originals, AdminUtils.fetchEntityConfig(replicaMgr.zkUtils,
        ConfigType.Topic, topicAndPartition.topic)).uncleanLeaderElectionEnable) {
        // Log a fatal error and shutdown the broker to ensure that data loss does not unexpectedly occur.
        fatal("...")
        Runtime.getRuntime.halt(1)
      }
```
调用`Runtime.getRuntime.halt(1)`直接暴力退出了.
可参考Kafka issue: [Unclean leader election and "Halting because log truncation is not allowed"](Unclean leader election and "Halting because log truncation is not allowed")
###### Replica从错误的Partition leader上去同步数据
- 现象: 集群里若干台机器先后磁盘空间报警, 经查是kafka log占用大量磁盘空间,接着看log, 里面有大量的
```
WARN [Replica Manager on Broker 3]: While recording the replica LEO, the partition [orderservice.production,0] hasn't been created. (kafka.server.ReplicaManager)
```
和
```
ERROR [ReplicaFetcherThread-0-58], Error for partition [reptest,0] to broker 58:org.apache.kafka.common.errors.UnknownTopicOrPartitionException: This server does not host this topic-partition. (kafka.server.ReplicaFetcherThread)
```
- 日志分析:
从上面的日志结合当前topic的partiton的复本和isr情况,可知是错误的`replica`从错误的`partition leader`上去同步数据了, 这理论上不应该啊;
  1. 之前每个集群因硬件原因挂掉了一台机器, 然后想删掉上面的一个partiton, 但因为kafka本身不支持partiton的删除, 就在zk上的`/brokers/[topic]`节点的内容里直接去掉了这个partiton的信息, 但是`kafka controller`并不会处理partiton减少的情况, 可参考[KafkaController分析](KafkaController分析7-启动流程)
  2. 为了触发这个topic的partition的删除, 我又迁移了其他的partiton;
  3. 然后还删除了zk上的`/controller`临时节点;
  4. 最后连自己都晕了;
  5. 然后之前坏的机器修好又上线了, 然后问题出现了;
- 解决: 将broker都重启了一遍;
- 深究: 
   1. 最终原因没有完全确认, 发现问题的时候之前的kafka debug log被删除了;
   2. kafka 上有类以的issue: [can't create as many partitions as brokers exists](can't create as many partitions as brokers exists)
   3. 尽量不要手动更新zk上的kafka相关节点内容;
   4. 考虑在kafka源码里加个delete partition的功能, 这个不会太难;
###### __consumer_offsets日志无法被清除
- 现象: 集群中若干台机器磁盘空间报警, 上去查看是__consumer_offsets的一个partition占用了几十G的空间
- 日志分析: 之前的日志被清理了,没有有效的日志了.为了debug这个问题,我把这个partition下的index和log文件打包拷贝到了测试集群, 然后重启了当前的broker, 发现了下面的日志:
```
[2017-09-30 10:49:36,126] ERROR [kafka-log-cleaner-thread-0], Error due to  (kafka.log.LogCleaner)
java.lang.IllegalArgumentException: requirement failed: 138296566648 messages in segment __consumer_offsets-5/00000000000000000000.log but offset map can fit only 5033164. You can increase log.cleaner.dedupe.buffer.size or decrease log.cleaner.threads
        at scala.Predef$.require(Predef.scala:219)
        at kafka.log.Cleaner$$anonfun$buildOffsetMap$4.apply(LogCleaner.scala:584)
        at kafka.log.Cleaner$$anonfun$buildOffsetMap$4.apply(LogCleaner.scala:580)
        at scala.collection.immutable.Stream$StreamWithFilter.foreach(Stream.scala:570)
        at kafka.log.Cleaner.buildOffsetMap(LogCleaner.scala:580)
        at kafka.log.Cleaner.clean(LogCleaner.scala:322)
        at kafka.log.LogCleaner$CleanerThread.cleanOrSleep(LogCleaner.scala:230)
        at kafka.log.LogCleaner$CleanerThread.doWork(LogCleaner.scala:208)
        at kafka.utils.ShutdownableThread.run(ShutdownableThread.scala:63)
```
- 问题分析:
   结合`LogCleaner`的源码可知,是`00000000000000000000.log`这个`logSegment`的`segment.nextOffset() - segment.baseOffset`大于了`maxDesiredMapSize`, 导致了`LogClean`线程的终止, 从而无法清理, 这不应该啊?! 
```
     val segmentSize = segment.nextOffset() - segment.baseOffset
      require(segmentSize <= maxDesiredMapSize, "%d messages in segment %s/%s but offset map can fit only %d. You can in了crease log.cleaner.dedupe.buffer.size or decrease log.cleaner.threads".format(segmentSize,  log.name, segment.log.file.getName, maxDesiredMapSize))
      if (map.size + segmentSize <= maxDesiredMapSize)
        offset = buildOffsetMapForSegment(log.topicAndPartition, segment, map)
      else
        full = true
```
- 解决: 我也没想到其他的好办法, 暴力删除了`00000000000000000000.log`和`00000000000000000000.index`, 然后删掉了`cleaner-offset-checkpoint`中相关的项,重启broker, 日志开始了压缩清理
- 深究:
  这个`logSegment`的`segment.nextOffset() - segment.baseOffset`大于了`maxDesiredMapSize`, 猜测是有个业务是手动提交offset到这个partition, 没有控制好,导致每秒能提交8,9MByte上来;
###### GC问题
- 现象: 集群报警某台broker down, 在zk上无此broker节点的注册信息
- 日志分析:
   1.  看broker日志里报zk连接超时或不能从zk读取任何数据, 其实和上面的**Broker到zk集群的连接不时会断开重断**现象是一样的;
  2. 看broker的gc日志, 对应时间gc耗时很长, 导致`stop the world`,broker所有线程都停止工作, 自然也无法与zk保持心跳;
- 解决: 暂时无解决方案, `GC`是个大麻烦, 网上也搜了一圈, 没找到有效的解决方案, 个人水平有限, 哪位大神有什么好的方法, 可以留言给我,谢谢~
- 补充: 关于GC这个找到了庄博士的这个视频,可以参考下[OS 造成的长时间非典型 JVM GC 停顿：深度分析和解决](OS 造成的长时间非典型 JVM GC 停顿：深度分析和解决 )
- GC慢,引起的STW会导致很多问题, 我们还遇到了他导致的OOM, Listen队列被打满
###### zk和kafka部署
*  zk和kafka broker 如果部署在同一台机器上, 请尽量将各自的data和log路径放在不同的磁盘, 避免磁盘io的竞争;
* kafka对zk的波动很敏感, 因此zk最好是单独部署,保证其稳定运行;
* 对zk不要有大量的写入操作, zk的写操作最后都会转移动leader上zk;
* 如果采用了zk和broker是混部的方式,并且还有大量的zk写入操作,比如使用较旧版本的storm,其提交offset到zk上, 导致zk的IO较高, 在启动zk时可以加上`zookeeper.forceSync=no`, 降低写盘IO, 这个配置有其副作用, 在线上使用时还需慎重;
###### 监控很重要
- 实时监控: 在集群上建立一个专门的topic, 监控程序实时的写入数据, 但无法写入或写入耗时达到阈值时报警, 这个实时监控真的真好用,基本上都第一时间发现问题;
- 基础监控: cpu, 磁盘IO, 网卡流量, FD, 连接数等;
- Topic流量监控: 监控topic的生产和消费流量, 特别是流量突增的情况, 快速找出害群之马, 可以通过kafka的jmx来获取相关的数据, 使用`Grafana`来显示和报警;
###### 大量异常: `Attempted to decrease connection count for address with no connections`
- 现象: 集群中某台broker所在机器磁盘报警, 查看是server.log很大;
- 日志分析: 日志里在刷大量的如下log:
```
[2016-10-13 00:00:00,495] ERROR Processor got uncaught exception. (kafka.network.Processor)
java.lang.IllegalArgumentException: Attempted to decrease connection count for address with no connections, address: /xxx.xxx.xxx.xxx
        at kafka.network.ConnectionQuotas$$anonfun$9.apply(SocketServer.scala:565)
        at kafka.network.ConnectionQuotas$$anonfun$9.apply(SocketServer.scala:565)
        at scala.collection.MapLike$class.getOrElse(MapLike.scala:128)
        at scala.collection.AbstractMap.getOrElse(Map.scala:59)
        at kafka.network.ConnectionQuotas.dec(SocketServer.scala:564)
        at kafka.network.Processor$$anonfun$run$13.apply(SocketServer.scala:450)
        at kafka.network.Processor$$anonfun$run$13.apply(SocketServer.scala:445)
        at scala.collection.Iterator$class.foreach(Iterator.scala:742)
        at scala.collection.AbstractIterator.foreach(Iterator.scala:1194)
        at scala.collection.IterableLike$class.foreach(IterableLike.scala:72)
        at scala.collection.AbstractIterable.foreach(Iterable.scala:54)
        at kafka.network.Processor.run(SocketServer.scala:445)
        at java.lang.Thread.run(Thread.java:745)
```
- 解决:
   1. 0.9.0.1的Bug, 打path: [Exception when attempting to decrease connection count for address with no connections](Exception when attempting to decrease connection count for address with no connections)
  2. 重启broker, 暂时性的解决方案;
###### 新版sdk访问较旧版的kafka, 发送kafka不支持的request
- 现象: 日志里有大量如下日志:
```
[2017-10-12 16:52:38,141] ERROR Processor got uncaught exception. (kafka.network.Processor)
java.lang.ArrayIndexOutOfBoundsException: 18
        at org.apache.kafka.common.protocol.ApiKeys.forId(ApiKeys.java:68)
        at org.apache.kafka.common.requests.AbstractRequest.getRequest(AbstractRequest.java:39)
        at kafka.network.RequestChannel$Request.<init>(RequestChannel.scala:79)
        at kafka.network.Processor$$anonfun$run$11.apply(SocketServer.scala:426)
        at kafka.network.Processor$$anonfun$run$11.apply(SocketServer.scala:421)
        at scala.collection.Iterator$class.foreach(Iterator.scala:742)
        at scala.collection.AbstractIterator.foreach(Iterator.scala:1194)
        at scala.collection.IterableLike$class.foreach(IterableLike.scala:72)
        at scala.collection.AbstractIterable.foreach(Iterable.scala:54)
        at kafka.network.Processor.run(SocketServer.scala:421)
        at java.lang.Thread.run(Thread.java:745)
```
- 分析: 
   1. 当前用的kafka版本为0.9.0.1, 支持的request最大id为16, 这个18是新版 kafka中的[ApiVersion Request](Apache Kafka), 因此会抛这个异常出来;
   2. 跟了一下代码, 在`SocketServer`中:
```
         try {
            val channel = selector.channel(receive.source)
            val session = RequestChannel.Session(new KafkaPrincipal(KafkaPrincipal.USER_TYPE, channel.principal.getName),
              channel.socketAddress)
            val req = RequestChannel.Request(processor = id, connectionId = receive.source, session = session, buffer = receive.payload, startTimeMs = time.milliseconds, securityProtocol = protocol)
            requestChannel.sendRequest(req)
          } catch {
            case e @ (_: InvalidRequestException | _: SchemaException) =>
              // note that even though we got an exception, we can assume that receive.source is valid. Issues with constructing a valid receive object were handled earlier
              error("Closing socket for " + receive.source + " because of error", e)
              isClose = true
              close(selector, receive.source)
          }
```
在处理Request时并未处理这个异常,导致这个异常被其外层的`try...catch...`处理, 直接进入了下一轮的`selector.poll(300)`, 而在这个`selector.poll(300)`中会清理之前所有的接收到的Requests, 这就导致在这种情况下,可能会漏处理一些Request, 这样看起来还是个比较严重的问题;
- 解决:
  1. 一个简单修复:
```
selector.completedReceives.asScala.foreach { receive =>
          var isClose = false

          try {
            val channel = selector.channel(receive.source)
            val session = RequestChannel.Session(new KafkaPrincipal(KafkaPrincipal.USER_TYPE, channel.principal.getName),
              channel.socketAddress)
            val req = RequestChannel.Request(processor = id, connectionId = receive.source, session = session, buffer = receive.payload, startTimeMs = time.milliseconds, securityProtocol = protocol)
            requestChannel.sendRequest(req)
          } catch {
            case e @ (_: InvalidRequestException | _: SchemaException) =>
              // note that even though we got an exception, we can assume that receive.source is valid. Issues with constructing a valid receive object were handled earlier
              error("Closing socket for " + receive.source + " because of error", e)
              isClose = true
              close(selector, receive.source)
            case e : ArrayIndexOutOfBoundsException =>
              error("NotSupport Request | Closing socket for " + receive.source + " because of error", e)
              isClose = true
              close(selector, receive.source)
          }
          if (!isClose) {
            selector.mute(receive.source)
          }
        }
```
  2. Kafka上也有相关的[Broker does not disconnect client on unknown request](Broker does not disconnect client on unknown request), 这个修复内容比较多. 

###### 频繁FullGC
- 现象: Kafka broker停止工作, 日志无输出,整个进程Hang住;
- 分析: 查看kafkaServer-gc.log, 有FullGC log, 内存无法回收, 考虑是存在内存泄漏
我们找到了 [SocketServer inflightResponses collection leaks memory on client disconnect](SocketServer inflightResponses collection leaks memory on client disconnect): `inflightResponses`会缓存住需要发送但还没有发送完成的response, 这个response又同时持有其对应的request的引用, 访问请求量大的时候其内存占用不少.
对于`inflightResponses`0.9.0.1代码中只在completedSends中作了remove, 在`disconnected`和`close`中没有处理;
- 修复:
   1. 最暴力的,可以直接将这个`inflightResponses`变量去掉, 但这会有个副作用,会影响到Metrics的统计;
   2. 优雅的,可以参考最新的kafka代码, 在`disconnected`和`close`也加入移除的操作;

# [Kafka源码分析-汇总](Kafka源码分析-Content Table)