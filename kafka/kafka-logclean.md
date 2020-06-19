* 这里说的日志,是指Kafka保存写入消息的文件;
* Kafka日志清除策略包括中间:
   1. 基于时间和大小的删除策略;
   2. Compact清理策略;
* 我们这里主要介绍基于Compact策略的Log Clean;
---
###### Compact策略说明
- Kafka官网介绍: [Log compaction](https://kafka.apache.org/documentation.html#compaction);
- Compact就是压缩, 只能针对特定的topic应用此策略,即写入的`message`都带有`Key`, 合并相同`Key`的`message`, 只留下最新的`message`;
- 在压缩过程中, 针对`message`的payload为`null`的也将会去除掉;
- 官网上扒了一张图, 大家先感受下:

![110.png](http://upload-images.jianshu.io/upload_images/2020390-22b97d5337e6235d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 日志清理过程中的状态
- 主要涉及三种状态: `LogCleaningInProgress`, `LogCleaningAborted`,和`LogCleaningPaused`, 从字面上就很容易理解是什么意思,下面是源码中的注释:
 >  * If a partition is to be cleaned, it enters the LogCleaningInProgress state.
  >  * While a partition is being cleaned, it can be requested to be aborted and paused. Then the partition first enters
>  * the LogCleaningAborted state. Once the cleaning task is aborted, the partition enters the LogCleaningPaused state.
>  * While a partition is in the LogCleaningPaused state, it won't be scheduled for cleaning again, until cleaning is requested to be resumed.

- `LogCleanerManager`类 管理所有清理的log的状态及转换:
```
def abortCleaning(topicAndPartition: TopicAndPartition)
def abortAndPauseCleaning(topicAndPartition: TopicAndPartition)
def resumeCleaning(topicAndPartition: TopicAndPartition)
def checkCleaningAborted(topicAndPartition: TopicAndPartition) 
```
###### 要清理的日志的选取
- 因为这个compact清理过程涉及到log和index等文件的重写,比较耗IO, 因此kafka会作流控, 每次compact时都会先按规则确定要清理哪些`TopicAndPartiton`的log;
- 使用`LogToClean`类来表示要被清理的Log:
```
private case class LogToClean(topicPartition: TopicAndPartition, log: Log, firstDirtyOffset: Long) extends Ordered[LogToClean] {
  val cleanBytes = log.logSegments(-1, firstDirtyOffset).map(_.size).sum
  val dirtyBytes = log.logSegments(firstDirtyOffset, math.max(firstDirtyOffset, log.activeSegment.baseOffset)).map(_.size).sum
  val cleanableRatio = dirtyBytes / totalBytes.toDouble
  def totalBytes = cleanBytes + dirtyBytes
  override def compare(that: LogToClean): Int = math.signum(this.cleanableRatio - that.cleanableRatio).toInt
}
```
  1. `firstDirtyOffset`:表示本次清理的起始点, 其前边的offset将被作清理,与在其后的`message`作`key`的合并;
  2. `val cleanableRatio = dirtyBytes / totalBytes.toDouble`, 需要清理的log的比例,这个值越大,越可能被最后选中作清理;
  3. 每次清理完,要更新当前已经清理到的位置, 记录在`cleaner-offset-checkpoint`文件中,作为下一次清理时生成`firstDirtyOffset`的参考;
```
def updateCheckpoints(dataDir: File, update: Option[(TopicAndPartition,Long)]) {
    inLock(lock) {
      val checkpoint = checkpoints(dataDir)
      val existing = checkpoint.read().filterKeys(logs.keys) ++ update
      checkpoint.write(existing)
    }
  }
```
- 选出最需要清理的日志:
```
def grabFilthiestLog(): Option[LogToClean] = {
    inLock(lock) {
      val lastClean = allCleanerCheckpoints()
      val dirtyLogs = logs.filter {
        case (topicAndPartition, log) => log.config.compact  // skip any logs marked for delete rather than dedupe
      }.filterNot {
        case (topicAndPartition, log) => inProgress.contains(topicAndPartition) // skip any logs already in-progress
      }.map {
        case (topicAndPartition, log) => // create a LogToClean instance for each
          // if the log segments are abnormally truncated and hence the checkpointed offset
          // is no longer valid, reset to the log starting offset and log the error event
          val logStartOffset = log.logSegments.head.baseOffset
          val firstDirtyOffset = {
            val offset = lastClean.getOrElse(topicAndPartition, logStartOffset)
            if (offset < logStartOffset) {
              error("Resetting first dirty offset to log start offset %d since the checkpointed offset %d is invalid."
                    .format(logStartOffset, offset))
              logStartOffset
            } else {
              offset
            }
          }
          LogToClean(topicAndPartition, log, firstDirtyOffset)
      }.filter(ltc => ltc.totalBytes > 0) // skip any empty logs

      this.dirtiestLogCleanableRatio = if (!dirtyLogs.isEmpty) dirtyLogs.max.cleanableRatio else 0
      // and must meet the minimum threshold for dirty byte ratio
      val cleanableLogs = dirtyLogs.filter(ltc => ltc.cleanableRatio > ltc.log.config.minCleanableRatio)
      if(cleanableLogs.isEmpty) {
        None
      } else {
        val filthiest = cleanableLogs.max
        inProgress.put(filthiest.topicPartition, LogCleaningInProgress)
        Some(filthiest)
      }
    }
  }
```
  代码看着多,实在比较简单:
  1. 从所有的Log中产生出 `LogToClean`对象列表;
  2. 从1中获得的`LogToClean`列表中过滤过`cleanableRatio`大于config中配置的清理比率的`LogToClean`;
  3. 从2中获取的`LogToClean`列表中取`cleanableRatio`最大的,即为当前最需要被清理的.
###### 先放两张网上扒来的图:

![111.png](http://upload-images.jianshu.io/upload_images/2020390-2825506284404402.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  1. 这里的`CleanerPoint`就是我们上面说的`firstDirtyOffset`;
  2. `Log Tail`中的key将被合并到 `LogHead`中,实际上因为构建`OffsetMap`是在`Log Head`部分,因此合并`Key`的部分还包括构建`OffsetMap`最后到达的`Offset`位置;

**下面这个是整个压缩合并的过程, Kafka的代码就是把这个过程翻译成Code**
![112.png](http://upload-images.jianshu.io/upload_images/2020390-a0f243247b6b94c5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
###### 构建OffsetMap
- 构建上面图111.png中`LogHead`部分的所有日志的OffsetMap, 此Map中的key即为`message.key`的hash值, value即为当前message的`offset`
- 实现:
```
private[log] def buildOffsetMap(log: Log, start: Long, end: Long, map: OffsetMap): Long = {
    map.clear()
    val dirty = log.logSegments(start, end).toSeq
    info("Building offset map for log %s for %d segments in offset range [%d, %d).".format(log.name, dirty.size, start, end))
    
    // Add all the dirty segments. We must take at least map.slots * load_factor,
    // but we may be able to fit more (if there is lots of duplication in the dirty section of the log)
    var offset = dirty.head.baseOffset
    require(offset == start, "Last clean offset is %d but segment base offset is %d for log %s.".format(start, offset, log.name))
    val maxDesiredMapSize = (map.slots * this.dupBufferLoadFactor).toInt
    var full = false
    for (segment <- dirty if !full) {
      checkDone(log.topicAndPartition)
      val segmentSize = segment.nextOffset() - segment.baseOffset

      require(segmentSize <= maxDesiredMapSize, "%d messages in segment %s/%s but offset map can fit only %d. You can increase log.cleaner.dedupe.buffer.size or decrease log.cleaner.threads".format(segmentSize,  log.name, segment.log.file.getName, maxDesiredMapSize))
      if (map.size + segmentSize <= maxDesiredMapSize)
        offset = buildOffsetMapForSegment(log.topicAndPartition, segment, map)
      else
        full = true
    }
    info("Offset map for log %s complete.".format(log.name))
    offset
  }
```
   1. 顺序读取每个[LogSegment](http://www.jianshu.com/p/e20449a396b1), 将相关信息put到`OffsetMap`, 其中的`key`是`message.key`的hash值, 这个地方有个坑,如果出现了hash碰撞怎么?
   2. build的OffsetMap有大小限制, 不能超过`val maxDesiredMapSize = (map.slots * this.dupBufferLoadFactor).toInt`.
###### 重新分组需要清理的LogSegments
- 因为压缩清理后,原来的单个`LogSegment`势必大小要减少,因此需要重新分组来为重写`Log`和`Index`文件作准备;
- 分组的规则也很简单: 根据`segmentsize`和`indexsize`进行分组,这个分组是每一组的`segmentsize`不能超过`segmentSize`的配置大小,`indexfile`不能超过配置的最大`indexsize`的大小,同时条数不能超过`int.maxvalue`.
```
private[log] def groupSegmentsBySize(segments: Iterable[LogSegment], maxSize: Int, maxIndexSize: Int): List[Seq[LogSegment]] = {
    var grouped = List[List[LogSegment]]()
    var segs = segments.toList
    while(!segs.isEmpty) {
      var group = List(segs.head)
      var logSize = segs.head.size
      var indexSize = segs.head.index.sizeInBytes
      segs = segs.tail
      while(!segs.isEmpty &&
            logSize + segs.head.size <= maxSize &&
            indexSize + segs.head.index.sizeInBytes <= maxIndexSize &&
            segs.head.index.lastOffset - group.last.index.baseOffset <= Int.MaxValue) {
        group = segs.head :: group
        logSize += segs.head.size
        indexSize += segs.head.index.sizeInBytes
        segs = segs.tail
      }
      grouped ::= group.reverse
    }
    grouped.reverse
  }
```
###### 按上面重新分成的组作真正的清理工作
- 清理的过程,遍历所有需要清理的`LogSegment`, 按一定的规则过滤出需要保留的msg重定入新的Log文件中;
- 符合下列规则的`message`将被保留
   1. `message`的`key`在`OffsetMap`中能找到,同时当前的`message`的`offset`不小于`offsetMap`中存储的`offset`;
   2. 这个`segment`的最后修改时间大于最大的保留时间,同时这个消息的`value`是有效的value,即不为null;
```
private def shouldRetainMessage(source: kafka.log.LogSegment,
                                  map: kafka.log.OffsetMap,
                                  retainDeletes: Boolean,
                                  entry: kafka.message.MessageAndOffset): Boolean = {
    val key = entry.message.key
    if (key != null) {
      val foundOffset = map.get(key)
      /* two cases in which we can get rid of a message:
       *   1) if there exists a message with the same key but higher offset
       *   2) if the message is a delete "tombstone" marker and enough time has passed
       */
      val redundant = foundOffset >= 0 && entry.offset < foundOffset
      val obsoleteDelete = !retainDeletes && entry.message.isNull
      !redundant && !obsoleteDelete
    } else {
      stats.invalidMessage()
      false
    }
  }
```
- 清理:
  1. 实现下就是按上面的规则过滤过需要保留的`message`后重写`Log`和`Index`文件过程;
  2. 具体写文件的流程可参考 [Kafka中Message存储相关类大揭密](http://www.jianshu.com/p/172a7a86be15)
```
private[log] def cleanSegments(log: Log,
                                 segments: Seq[LogSegment], 
                                 map: OffsetMap, 
                                 deleteHorizonMs: Long) {
    // create a new segment with the suffix .cleaned appended to both the log and index name
    val logFile = new File(segments.head.log.file.getPath + Log.CleanedFileSuffix)
    logFile.delete()
    val indexFile = new File(segments.head.index.file.getPath + Log.CleanedFileSuffix)
    indexFile.delete()
    val messages = new FileMessageSet(logFile, fileAlreadyExists = false, initFileSize = log.initFileSize(), preallocate = log.config.preallocate)
    val index = new OffsetIndex(indexFile, segments.head.baseOffset, segments.head.index.maxIndexSize)
    val cleaned = new LogSegment(messages, index, segments.head.baseOffset, segments.head.indexIntervalBytes, log.config.randomSegmentJitter, time)

    try {
      // clean segments into the new destination segment
      for (old <- segments) {
        val retainDeletes = old.lastModified > deleteHorizonMs
        info("Cleaning segment %s in log %s (last modified %s) into %s, %s deletes."
            .format(old.baseOffset, log.name, new Date(old.lastModified), cleaned.baseOffset, if(retainDeletes) "retaining" else "discarding"))
        cleanInto(log.topicAndPartition, old, cleaned, map, retainDeletes)
      }

      // trim excess index
      index.trimToValidSize()

      // flush new segment to disk before swap
      cleaned.flush()

      // update the modification date to retain the last modified date of the original files
      val modified = segments.last.lastModified
      cleaned.lastModified = modified

      // swap in new segment
      info("Swapping in cleaned segment %d for segment(s) %s in log %s.".format(cleaned.baseOffset, segments.map(_.baseOffset).mkString(","), log.name))
      log.replaceSegments(cleaned, segments)
    } catch {
      case e: LogCleaningAbortedException =>
        cleaned.delete()
        throw e
    }
  }
```