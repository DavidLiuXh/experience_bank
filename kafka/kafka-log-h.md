* Kafka里有关log操作的类比较类, 但是层次关系还是很清晰的,实际上就是上次会把操作代理给下一层;
* 是时候放出这张图了

![Log层级.png](http://upload-images.jianshu.io/upload_images/2020390-3414b44a548d2abd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 相关的一些类我们在前面的章节中都有介绍过
  1. [Kafka的日志管理模块--LogManager](http://www.jianshu.com/p/b23584ab1419)
  2. [Kafka中Message存储相关类大揭密](http://www.jianshu.com/p/172a7a86be15)
  3. [Kafka消息的磁盘存储](http://www.jianshu.com/p/e20449a396b1)
* 目前看起来我们只剩下上图中的`Log`类没有介绍, 所以这章基本上就是过一下这个`Log`类
---
###### Log
- 所在文件: core/src/main/scala/kafka/log/Log.scala
- 作用: kafka的数据落盘存在不同的目录下,目录的命名规则是`Topic-Partiton`, 这个`Log`封装的就是针对这样的每个目录的操作
- 主要方法:
  1. `private val segments: ConcurrentNavigableMap[java.lang.Long, LogSegment] = new ConcurrentSkipListMap[java.lang.Long, LogSegment]`: 每个目录里包含多个`LogSegment`, 每个Segment分为Log和Index两类文件,这两个文件的以存储的最小的offset来命名,这个Map管理了当前目录下所有的`LogSegment`, key就是这个最小的offset;
  2. `private def loadSegments()`: 从磁盘文件加载初始化每个`LogSegment`, 在每个`Log`类对象创建初始化时会调用, 这个函数比较重要, 下面的代码里加了注释
```
    dir.mkdirs()
    var swapFiles = Set[File]()
    
    // first do a pass through the files in the log directory and remove any temporary files 
    // and find any interrupted swap operations
    for(file <- dir.listFiles if file.isFile) {
      if(!file.canRead)
        throw new IOException("Could not read file " + file)
      val filename = file.getName
      //对于.deleted和.cleaned结尾的文件直接删除
      if(filename.endsWith(DeletedFileSuffix) || filename.endsWith(CleanedFileSuffix)) {
        // if the file ends in .deleted or .cleaned, delete it
        file.delete()
      } else if(filename.endsWith(SwapFileSuffix)) {
        // we crashed in the middle of a swap operation, to recover:
        // if a log, delete the .index file, complete the swap operation later
        // if an index just delete it, it will be rebuilt

       //.swap文件需要真正恢复, 对应的indes文件都删除
        val baseName = new File(CoreUtils.replaceSuffix(file.getPath, SwapFileSuffix, ""))
        if(baseName.getPath.endsWith(IndexFileSuffix)) {
          file.delete()
        } else if(baseName.getPath.endsWith(LogFileSuffix)){
          // delete the index
          val index = new File(CoreUtils.replaceSuffix(baseName.getPath, LogFileSuffix, IndexFileSuffix))
          index.delete()
          swapFiles += file
        }
      }
    }

    // now do a second pass and load all the .log and .index files
    for(file <- dir.listFiles if file.isFile) {
      val filename = file.getName
      if(filename.endsWith(IndexFileSuffix)) {
        //有index文件但没有对应的log文件,则删除index文件
        // if it is an index file, make sure it has a corresponding .log file
        val logFile = new File(file.getAbsolutePath.replace(IndexFileSuffix, LogFileSuffix))
        if(!logFile.exists) {
          warn("Found an orphaned index file, %s, with no corresponding log file.".format(file.getAbsolutePath))
          file.delete()
        }
      } else if(filename.endsWith(LogFileSuffix)) {
        // if its a log file, load the corresponding log segment
        val start = filename.substring(0, filename.length - LogFileSuffix.length).toLong
        val indexFile = Log.indexFilename(dir, start)
        val segment = new LogSegment(dir = dir, 
                                     startOffset = start,
                                     indexIntervalBytes = config.indexInterval, 
                                     maxIndexSize = config.maxIndexSize,
                                     rollJitterMs = config.randomSegmentJitter,
                                     time = time,
                                     fileAlreadyExists = true)

        if(indexFile.exists()) {
          try {
              segment.index.sanityCheck()
          } catch {
            case e: java.lang.IllegalArgumentException =>
              warn("Found a corrupted index file, %s, deleting and rebuilding index...".format(indexFile.getAbsolutePath))
              indexFile.delete()
              segment.recover(config.maxMessageSize)
          }
        }
        else {
          error("Could not find index file corresponding to log file %s, rebuilding index...".format(segment.log.file.getAbsolutePath))
          segment.recover(config.maxMessageSize)
        }
        segments.put(start, segment)
      }
    }
    
    // 针对.swap文件作恢复,实际上就是删除目录下对swap文件的offset有重叠的log文件
    // Finally, complete any interrupted swap operations. To be crash-safe,
    // log files that are replaced by the swap segment should be renamed to .deleted
    // before the swap file is restored as the new segment file.
    for (swapFile <- swapFiles) {
      val logFile = new File(CoreUtils.replaceSuffix(swapFile.getPath, SwapFileSuffix, ""))
      val fileName = logFile.getName
      val startOffset = fileName.substring(0, fileName.length - LogFileSuffix.length).toLong
      val indexFile = new File(CoreUtils.replaceSuffix(logFile.getPath, LogFileSuffix, IndexFileSuffix) + SwapFileSuffix)
      val index =  new OffsetIndex(file = indexFile, baseOffset = startOffset, maxIndexSize = config.maxIndexSize)
      val swapSegment = new LogSegment(new FileMessageSet(file = swapFile),
                                       index = index,
                                       baseOffset = startOffset,
                                       indexIntervalBytes = config.indexInterval,
                                       rollJitterMs = config.randomSegmentJitter,
                                       time = time)
      info("Found log file %s from interrupted swap operation, repairing.".format(swapFile.getPath))
      swapSegment.recover(config.maxMessageSize)
      val oldSegments = logSegments(swapSegment.baseOffset, swapSegment.nextOffset)
      replaceSegments(swapSegment, oldSegments.toSeq, isRecoveredSwapFile = true)
    }

    if(logSegments.size == 0) {
      // no existing segments, create a new mutable segment beginning at offset 0
      segments.put(0L, new LogSegment(dir = dir,
                                     startOffset = 0,
                                     indexIntervalBytes = config.indexInterval, 
                                     maxIndexSize = config.maxIndexSize,
                                     rollJitterMs = config.randomSegmentJitter,
                                     time = time,
                                     fileAlreadyExists = false,
                                     initFileSize = this.initFileSize(),
                                     preallocate = config.preallocate))
    } else {
      recoverLog()
      // reset the index size of the currently active log segment to allow more entries
      activeSegment.index.resize(config.maxIndexSize)
    }
```
  3. `def append(messages: ByteBufferMessageSet, assignOffsets: Boolean = true)` : 追加新的msg到Log文件
```
     3.1   对`messages`中的每条`Record`重新赋予offset
          val offset = new AtomicLong(nextOffsetMetadata.messageOffset)
          try {
            validMessages = validMessages.validateMessagesAndAssignOffsets(offset, appendInfo.sourceCodec, appendInfo.targetCodec, config.compact)
          } catch {
            case e: IOException => throw new KafkaException("Error in validating messages while appending to log '%s'".format(name), e)

      3.2  验证每条`Record`中的msg大小是否超出系统配置中的限制
           for(messageAndOffset <- validMessages.shallowIterator) {
          if(MessageSet.entrySize(messageAndOffset.message) > config.maxMessageSize) {
            // we record th   e original message set size instead of trimmed size
            // to be consistent with pre-compression bytesRejectedRate recording
            BrokerTopicStats.getBrokerTopicStats(topicAndPartition.topic).bytesRejectedRate.mark(messages.sizeInBytes)
            BrokerTopicStats.getBrokerAllTopicsStats.bytesRejectedRate.mark(messages.sizeInBytes)
            throw new MessageSizeTooLargeException("Message size is %d bytes which exceeds the maximum configured message size of %d."
              .format(MessageSet.entrySize(messageAndOffset.message), config.maxMessageSize))
          }
        }

      3.3 检查Record set的整体大小是否超出一个LogSegment的配置限制
            if(validMessages.sizeInBytes > config.segmentSize) {
          throw new MessageSetSizeTooLargeException("Message set size is %d bytes which exceeds the maximum configured segment size of %d."
            .format(validMessages.sizeInBytes, config.segmentSize))
        }

     3.4 如果需要的话，关闭当前的LogSegment, 新建一个LogSegment用入写入当前的msg
            val segment = maybeRoll(validMessages.sizeInBytes)

     3.5 追加新msg到ActiveLogSegment
           segment.append(appendInfo.firstOffset, validMessages)

     3.6 更新LogEndOffset
            updateLogEndOffset(appendInfo.lastOffset + 1)
```
4. `def read(startOffset: Long, maxLength: Int, maxOffset: Option[Long] = None)`： 从log文件中读取msg
```
   // **验证startOffset的有效性**
    val currentNextOffsetMetadata = nextOffsetMetadata
    val next = currentNextOffsetMetadata.messageOffset
    if(startOffset == next)
      return FetchDataInfo(currentNextOffsetMetadata, MessageSet.Empty)

    //锁定开始读取的LogSegment  
    var entry = segments.floorEntry(startOffset)

    // attempt to read beyond the log end offset is an error
    if(startOffset > next || entry == null)
      throw new OffsetOutOfRangeException("Request for offset %d but we only have log segments in the range %d to %d.".format(startOffset, segments.firstKey, next))

// 确定maxPosition
val maxPosition = {
        if (entry == segments.lastEntry) {
          val exposedPos = nextOffsetMetadata.relativePositionInSegment.toLong
          // Check the segment again in case a new segment has just rolled out.
          if (entry != segments.lastEntry)
            // New log segment has rolled out, we can read up to the file end.
            entry.getValue.size
          else
            exposedPos
        } else {
          entry.getValue.size
        }
      }

//读取
val fetchInfo = entry.getValue.read(startOffset, maxOffset, maxLength, maxPosition)
```