* 发送到Kafka的消息最终都是要落盘存储到磁盘上;
* 本章涉及到的类:
  1. OffsetIndex;
  2. LogSegment;
---
###### OffsetIndex类
- **所在文件**: core/src/main/scala/kafka/log/OffsetIndex.scala
- **作用**:  我们知道所有发送到kafka的消息都是以`Record`的结构([Kafka中Message存储相关类大揭密](http://www.jianshu.com/p/172a7a86be15))写入到本地文件, 有写就要有读,读取时一般是从给定的offset开始读取,这个offset是逻辑offset, 需要转换成文件的实际偏移量, 为了加速这个转换, kafka针对每个log文件,提供了index文件, index文件采用稀疏索引的方式, 只记录部分log offset到file position的转换, 然后还需要在log文件中进行少量的顺序遍历, 来精确定位到需要的`Record`;
- **index文件结构**: 文件里存的是一条条的log offset与file position的映射, 每条记录8个字节,前4个字节是log offset, 后4个字节是file position, 这样的每一条映射信息我们可以称为是一个**slot**
- **读写方式**: 为了加速index文件的读写, 采用了文件内存映射的方式:
```
    /* initialize the memory mapping for this index */
    private var mmap: MappedByteBuffer = 
    {
      val newlyCreated = file.createNewFile()
      val raf = new RandomAccessFile(file, "rw")
      try {
        /* pre-allocate the file if necessary */
        if(newlyCreated) {
          if(maxIndexSize < 8)
            throw new IllegalArgumentException("Invalid max index size: " + maxIndexSize)
          raf.setLength(roundToExactMultiple(maxIndexSize, 8))
        }
          
        /* memory-map the file */
        val len = raf.length()
        val idx = raf.getChannel.map(FileChannel.MapMode.READ_WRITE, 0, len)
          
        /* set the position in the index for the next entry */
        if(newlyCreated)
          idx.position(0)
        else
          // if this is a pre-existing index, assume it is all valid and set position to last entry
          idx.position(roundToExactMultiple(idx.limit, 8))
        idx
      } finally {
        CoreUtils.swallow(raf.close())
      }
    }

```
- 主要方法:
  1. `def lookup(targetOffset: Long): OffsetPosition`: 查找小于或等于`targetOffset`的最大Offset;
```
 maybeLock(lock) {
      val idx = mmap.duplicate
      val slot = indexSlotFor(idx, targetOffset)
      if(slot == -1)
        OffsetPosition(baseOffset, 0)
      else
        OffsetPosition(baseOffset + relativeOffset(idx, slot), physical(idx, slot))
      }
```
 2. `private def indexSlotFor(idx: ByteBuffer, targetOffset: Long): Int`:采用二分法查找到对于`targetOffset`在index文件中的**slot**
```
   // binary search for the entry 二分查找法
    var lo = 0
    var hi = entries-1
    while(lo < hi) {
      val mid = ceil(hi/2.0 + lo/2.0).toInt
      val found = relativeOffset(idx, mid)
      if(found == relOffset)
        return mid
      else if(found < relOffset)
        lo = mid
      else
        hi = mid - 1
    }
```
 3. `def append(offset: Long, position: Int)`: 向index文件中追加一个offset/location的映射信息
 4. `def truncateTo(offset: Long)`: 按给定的offset,找到对应的slot, 然后截断
 5. `def resize(newSize: Int)`: 重新设置index文件size, 但保持当前mmap的position不变
```
      inLock(lock) {
      val raf = new RandomAccessFile(file, "rw")
      val roundedNewSize = roundToExactMultiple(newSize, 8)
      val position = this.mmap.position
      
      /* Windows won't let us modify the file length while the file is mmapped :-( */
      if(Os.isWindows)
        forceUnmap(this.mmap)
      try {
        raf.setLength(roundedNewSize)
        this.mmap = raf.getChannel().map(FileChannel.MapMode.READ_WRITE, 0, roundedNewSize)
        this.maxEntries = this.mmap.limit / 8
        this.mmap.position(position)
      } finally {
        CoreUtils.swallow(raf.close())
      }
    }
```
- 有意思的一件事:
  上面我们说过这个index文件的读取是使用了内存文件映射`MappedByteBuffer`, 然后并没有找到相应的unmap(实际上是没有这方法)的调用, 这个会不会有问题呢?遇到google了一下, 果然有发现: [Long GC pause harming broker performance which is caused by mmap objects created for OffsetIndex](https://issues.apache.org/jira/browse/KAFKA-4614),
在实际应用中确实遇到了这样的问题,一直没搞明白为什么IO会升高.
###### LogSegment
- 所在文件: core/src/main/scala/kafka/log/LogSegment.scala
- 作用: 封装对消息落地后的log和index文件的所有操作
- 类定义:
```
      class LogSegment(val log: FileMessageSet, 
                 val index: OffsetIndex, 
                 val baseOffset: Long, 
                 val indexIntervalBytes: Int,
                 val rollJitterMs: Long,
                 time: Time) extends Loggin
```
可以看到使用[FileMessageSet](http://www.jianshu.com/p/172a7a86be15)来操作Log文件, 使用`OffsetIndex`来操作Index文件
- 主要方法:
  1. `def size: Long = log.sizeInBytes()` :   返回当前log文件的大小
  2. `def append(offset: Long, messages: ByteBufferMessageSet)`:追加msg到log文件尾部,必要时更新index文件
```
 if (messages.sizeInBytes > 0) {
      // append an entry to the index (if needed)
      // index采用的是稀疏索引, 所以先判断是否需要写入
      if(bytesSinceLastIndexEntry > indexIntervalBytes) {
        index.append(offset, log.sizeInBytes())
        this.bytesSinceLastIndexEntry = 0
      }
      // append the messages
      log.append(messages)  //追加msg到log文件尾部
      this.bytesSinceLastIndexEntry += messages.sizeInBytes
    }
```
 3. `def read(startOffset: Long, maxOffset: Option[Long], maxSize: Int, maxPosition: Long = size): FetchDataInfo`: 根据给定的offset信息等读取相应的msg 和offset信息,构成`FetchDataInfo`返回
```
 val offsetMetadata = new LogOffsetMetadata(startOffset, this.baseOffset, startPosition.position)

    // if the size is zero, still return a log segment but with zero size
    if(maxSize == 0)
      return FetchDataInfo(offsetMetadata, MessageSet.Empty)

    // calculate the length of the message set to read based on whether or not they gave us a maxOffset
    val length = 
      maxOffset match {
        case None =>
          // no max offset, just read until the max position
          min((maxPosition - startPosition.position).toInt, maxSize)
        case Some(offset) => {
          // there is a max offset, translate it to a file position and use that to calculate the max read size
          if(offset < startOffset)
            throw new IllegalArgumentException("Attempt to read with a maximum offset (%d) less than the start offset (%d).".format(offset, startOffset))
          val mapping = translateOffset(offset, startPosition.position)
          val endPosition = 
            if(mapping == null)
              logSize // the max offset is off the end of the log, use the end of the file
            else
              mapping.position
          min(min(maxPosition, endPosition) - startPosition.position, maxSize).toInt
        }
      }
    FetchDataInfo(offsetMetadata, log.read(startPosition.position, length))
```
实际上最终是调用`FileMessageSet`的`read`方法读取
 4. `def recover(maxMessageSize: Int): Int` :读取当前的log文件内容,重新构建index文件
 ```
 //逐条读取log里的msg, 然后构建index文件
 val iter = log.iterator(maxMessageSize)
    try {
      while(iter.hasNext) {
        val entry = iter.next
        entry.message.ensureValid()
        if(validBytes - lastIndexEntry > indexIntervalBytes) {
          // we need to decompress the message, if required, to get the offset of the first uncompressed message
          val startOffset =
            entry.message.compressionCodec match {
              case NoCompressionCodec =>
                entry.offset
              case _ =>
                ByteBufferMessageSet.deepIterator(entry.message).next().offset
          }
          index.append(startOffset, validBytes)
          lastIndexEntry = validBytes
        }
        validBytes += MessageSet.entrySize(entry.message)
      }
    } catch {
      case e: InvalidMessageException => 
        logger.warn("Found invalid messages in log segment %s at byte offset %d: %s.".format(log.file.getAbsolutePath, validBytes, e.getMessage))
    }
 ```
 5. `def truncateTo(offset: Long): Int`: 根据给定的offset截断log和index文件
```
 val mapping = translateOffset(offset)
    if(mapping == null)
      return 0
    index.truncateTo(offset)
    // after truncation, reset and allocate more space for the (new currently  active) index
    index.resize(index.maxIndexSize)
    val bytesTruncated = log.truncateTo(mapping.position)
    if(log.sizeInBytes == 0)
      created = time.milliseconds
    bytesSinceLastIndexEntry = 0
    bytesTruncated
```
 6. `def nextOffset(): Long` : 获取下一个offset值, 其实就是当前最大的offset + 1
```
val ms = read(index.lastOffset, None, log.sizeInBytes)
    if(ms == null) {
      baseOffset
    } else {
      ms.messageSet.lastOption match {
        case None => baseOffset
        case Some(last) => last.nextOffset
      }
    }
```