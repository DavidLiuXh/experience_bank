* 这里说的日志不是为了追踪程序运行而打的日志，指的是Kafka接受到消息后将消息写入磁盘或从磁盘读取的子系统;
* 它负责Log的创建，遍历，清理，读写等;
* LogManager统领所有的Log对象, 具体的读写操作还是要转给Log对象,Log对象又包含若干个LogSegment, 一层套一层,逐层分解;
* 它支持将本地的多个文件夹作出日志的存储目录;
---
###### LogManager
* 所在文件：core/src/main/scala/kafka/log/LogManager.scala
* LogManager的创建：
  1. 在KafkaServer启动时创建，通过调用 `KafkaServer.createLogManager实现。
 2. 每个Topic都可以单独设置自己Log的过期时间，roll大小等，这些信息存储在zk上，因此集群管理员可以通过调整zk上的相应配置，在不重启整个集群的前提下，动态调整这些信息;
* LogManager的初始化:
 1. `private val logs = new Pool[TopicAndPartition, Log]()`: 使用`Pool`管理所有的Log对象;
 2. `createAndValidateLogDirs(logDirs)`: 目前支持将本地的多个文件夹作出日志的存储目录，因为需要创建和验证这些目录的有效性, 我们来看下是如何作的：
```
if(dirs.map(_.getCanonicalPath).toSet.size < dirs.size)
      throw new KafkaException("Duplicate log directory found: " + logDirs.mkString(", "))
    for(dir <- dirs) {
      if(!dir.exists) {
        info("Log directory '" + dir.getAbsolutePath + "' not found, creating it.")
        val created = dir.mkdirs()
        if(!created)
          throw new KafkaException("Failed to create data directory " + dir.getAbsolutePath)
      }
      if(!dir.isDirectory || !dir.canRead)
        throw new KafkaException(dir.getAbsolutePath + " is not a readable log directory.")
    }
```
判断是否有重复的log目录; 目录如不存在,则创建; 目录是否可读;

 3. `val dirLocks = lockLogDirs(logDirs)`:使用文件锁锁定目录
```
dirs.map { dir =>
      val lock = new FileLock(new File(dir, LockFile))
      if(!lock.tryLock())
        throw new KafkaException("Failed to acquire lock on file .lock in " + lock.file.getParentFile.getAbsolutePath + 
                               ". A Kafka instance in another process or thread is using this directory.")
      lock
    }
```
 4. `recoveryPointCheckpoints = logDirs.map(dir => (dir, new OffsetCheckpoint(new File(dir, RecoveryPointCheckpointFile)))).toMap`: 创建每个目录中的`recovery-point-offset-checkpoint`文件(这个文件里记录的各个offset之前的数据均已落盘成功)的读取类对象;
 5. `def loadLogs(): Unit`: 恢复并且加载日志目录中的日志文件, 针对每个LogDir分别处理
```
val threadPools = mutable.ArrayBuffer.empty[ExecutorService]
    val jobs = mutable.Map.empty[File, Seq[Future[_]]]

    for (dir <- this.logDirs) {
      val pool = Executors.newFixedThreadPool(ioThreads)
      threadPools.append(pool)

      val cleanShutdownFile = new File(dir, Log.CleanShutdownFile)

      if (cleanShutdownFile.exists) {
        debug(
          "Found clean shutdown file. " +
          "Skipping recovery for all logs in data directory: " +
          dir.getAbsolutePath)
      } else {
        // log recovery itself is being performed by `Log` class during initialization
        brokerState.newState(RecoveringFromUncleanShutdown)
      }

      var recoveryPoints = Map[TopicAndPartition, Long]()
      try {
        recoveryPoints = this.recoveryPointCheckpoints(dir).read
      } catch {
        case e: Exception => {
          warn("Error occured while reading recovery-point-offset-checkpoint file of directory " + dir, e)
          warn("Resetting the recovery checkpoint to 0")
        }
      }

      val jobsForDir = for {
        dirContent <- Option(dir.listFiles).toList
        logDir <- dirContent if logDir.isDirectory
      } yield {
        CoreUtils.runnable {
          debug("Loading log '" + logDir.getName + "'")

          val topicPartition = Log.parseTopicPartitionName(logDir)
          val config = topicConfigs.getOrElse(topicPartition.topic, defaultConfig)
          val logRecoveryPoint = recoveryPoints.getOrElse(topicPartition, 0L)

          val current = new Log(logDir, config, logRecoveryPoint, scheduler, time)
          val previous = this.logs.put(topicPartition, current)

          if (previous != null) {
            throw new IllegalArgumentException(
              "Duplicate log directories found: %s, %s!".format(
              current.dir.getAbsolutePath, previous.dir.getAbsolutePath))
          }
        }
      }

      jobs(cleanShutdownFile) = jobsForDir.map(pool.submit).toSeq
    }
    try {
      for ((cleanShutdownFile, dirJobs) <- jobs) {
        dirJobs.foreach(_.get)
        cleanShutdownFile.delete()
      }
    } catch {
      case e: ExecutionException => {
      }
    } finally {
      threadPools.foreach(_.shutdown())
    }
    info("Logs loading complete.")
  }
```
 a. 如果kafka进程是优雅干净地退出的,会创建一个名为**.kafka_cleanshutdown**的文件作为标识;
 b. 启动kafka时, 如果不存在该文件, 则broker的状态进入到
     **RecoveringFromUncleanShutdown**
 c. 针对dir下的每个topic子目录, 创建**Log**对象, 此对象在创建过程中会加载,恢复实际的消息, 每个这样的过程跑在一个使用**CoreUtils.runnable **创建的Job里, job再提交到线程池执行, 实际上是生成一个Feture,
 d. 等待c中所有的job都执行完, 以便完成所有的log加载,恢复过程;

 6. `def startup()`: 启动一个LogManager, 实际上是启动若干个定时任务:
```
scheduler.schedule("kafka-log-retention", 
                         cleanupLogs, 
                         delay = InitialTaskDelayMs, 
                         period = retentionCheckMs, 
                         TimeUnit.MILLISECONDS)
      scheduler.schedule("kafka-log-flusher", 
                         flushDirtyLogs, 
                         delay = InitialTaskDelayMs, 
                         period = flushCheckMs, 
                         TimeUnit.MILLISECONDS)
      scheduler.schedule("kafka-recovery-point-checkpoint",
                         checkpointRecoveryPointOffsets,
                         delay = InitialTaskDelayMs,
                         period = flushCheckpointMs,
                         TimeUnit.MILLISECONDS)
    }
    if(cleanerConfig.enableCleaner)
      cleaner.startup()
```
 我们来过一遍:
 a. `checkpointRecoveryPointOffsets`: 将每个Topic-Partition的recovery-point(这个值就是已经落盘的offset值,因为有些log可能还在pagecache里,没有落盘)写入到recovery-point文件;
 b. `flushDirtyLogs`: 针对每一个Log对象,如果flush时间到,就调用**log->flush**, 将pagecache中的消息落盘;
 c. `cleanupLogs`: 针对清除策略是删除而不是压缩的Log, 依照时间和文件大小作清理:
```
 for(log <- allLogs; if !log.config.compact) {
      debug("Garbage collecting '" + log.name + "'")
      total += cleanupExpiredSegments(log) + cleanupSegmentsToMaintainSize(log)
    }
```
 e. `cleaner.startup()`: 对日志不是删除, 而是采取压缩策略, 后面会专门讲下这个;
 7. `def createLog(topicAndPartition: TopicAndPartition, config: LogConfig): Log`: 创建Log对象, 使用`nextLogDir()`来选取当前log所在的目录;
 8. `def deleteLog(topicAndPartition: TopicAndPartition) `: 移除Log;
 9. ` def truncateTo(partitionAndOffsets: Map[TopicAndPartition, Long]) `: 截取log到指定的offset, 同时写recovery-point文件:
```
for ((topicAndPartition, truncateOffset) <- partitionAndOffsets) {
      val log = logs.get(topicAndPartition)
      // If the log does not exist, skip it
      if (log != null) {
        //May need to abort and pause the cleaning of the log, and resume after truncation is done.
        val needToStopCleaner: Boolean = (truncateOffset < log.activeSegment.baseOffset)
        if (needToStopCleaner && cleaner != null)
          cleaner.abortAndPauseCleaning(topicAndPartition)
        log.truncateTo(truncateOffset)
        if (needToStopCleaner && cleaner != null)
          cleaner.resumeCleaning(topicAndPartition)
      }
    }
    checkpointRecoveryPointOffsets()
```

###### 大致LogManager的内容就这么多