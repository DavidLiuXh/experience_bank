* 在正式开始扒代码之前, 先来个开胃菜,简单介绍一下kafka的基础组件和一些代码实现中用到的基础类库
***
# Kafka基础组件概述
* KafkaServer是整个Kafka的核心组件，里面包含了kafka对外提供功能的所有角色；
* 一图顶千言:

![kafkaserver1.png](http://upload-images.jianshu.io/upload_images/2020390-03297ae880cad595.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# Kafka辅助类库简介
## KafkaScheduler
* 所在文件: core/src/main/scala/kafka/utils/KafkaScheduler.scala
* 功能: 接收需周期性执行的任务和延迟作务的添加, 使用一组thread pool来执行具体的任务;
* 实现: 封装了 [java.util.concurrent.ScheduledThreadPoolExecutor](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ScheduledThreadPoolExecutor.html);
* 接口(原有注释已经很清晰):
```
/**
   * Initialize this scheduler so it is ready to accept scheduling of tasks
   */
  def startup()
  
  /**
   * Shutdown this scheduler. When this method is complete no more executions of background tasks will occur. 
   * This includes tasks scheduled with a delayed execution.
   */
  def shutdown()
  
  /**
   * Check if the scheduler has been started
   */
  def isStarted: Boolean
  
  /**
   * Schedule a task
   * @param name The name of this task
   * @param delay The amount of time to wait before the first execution
   * @param period The period with which to execute the task. If < 0 the task will execute only once.
   * @param unit The unit for the preceding times.
   */
  def schedule(name: String, fun: ()=>Unit, delay: Long = 0, period: Long = -1, unit: TimeUnit = TimeUnit.MILLISECONDS)
```

## ZkUtils
* 所在文件: core/scr/main/scala/kafka/utils/ZkUtils.scala
* 功能: 封装了可能用到的对zk上节点的创建,读,写,解析(主要是json)操作;
* 实现: 使用了一个小众的类库 [I0Itec](https://github.com/sgroschupf/zkclient) 来操作zk;
* 涉及到以下zk节点:
```
  val ConsumersPath = "/consumers"
  val BrokerIdsPath = "/brokers/ids"
  val BrokerTopicsPath = "/brokers/topics"
  val ControllerPath = "/controller"
  val ControllerEpochPath = "/controller_epoch"
  val ReassignPartitionsPath = "/admin/reassign_partitions"
  val DeleteTopicsPath = "/admin/delete_topics"
  val PreferredReplicaLeaderElectionPath = "/admin/preferred_replica_election"
  val BrokerSequenceIdPath = "/brokers/seqid"
  val IsrChangeNotificationPath = "/isr_change_notification"
  val EntityConfigPath = "/config"
  val EntityConfigChangesPath = "/config/changes"
```

## Pool
* 所在文件: core/src/main/scala/kafka/utils/Pool.scala
* 功能: 简单的并发对象池;
* 实现: 对ConcurrentHashMap的封裝;
* getAndMaybePut实现小技巧, 使用了double check技术, 在有值的情况下降低锁的开销;
```
def getAndMaybePut(key: K) = {
    if (valueFactory.isEmpty)
      throw new KafkaException("Empty value factory in pool.")
    val curr = pool.get(key)
    if (curr == null) {
      createLock synchronized {
        val curr = pool.get(key)
        if (curr == null)
          pool.put(key, valueFactory.get(key))
        pool.get(key)
      }
    }
    else
      curr
  }
```

# Logging
* 所在文件: core/src/main/scala/kafka/utils/Logging.scala
* 功能: 定义了`trait Logging` 供其他类继承,方便写日志;
* 实现: 对`org.apache.log4j.Logger`的封装;

# FileLock
* 所在文件: core/src/main/scala/kafka/utils/FileLock.scala
* 功能: 文件锁, 相当于linux的`/usr/bin/lockf`;
* 实现: 使用`java.nio.channels.FileLock`实现;

# ByteBounderBlockingQueue
* 所在文件: core/src/main/scala/kafkak/utils/ByteBoundedBlockingQueue.scala;
* 功能: 阻塞队列, 队列满的衡量标准有两条: 队列内元素个数达到了上限, 队列内所有元素的size之各达到了上限;
* 实现: 使用`java.util.concurrent.LinkedBlockingQueue`实现, 加上了对队列内已有元素size大小的check;
*  接口:
```
def offer(e: E, timeout: Long, unit: TimeUnit = TimeUnit.MICROSECONDS): Boolean
def offer(e: E): Boolean
def put(e: E): Boolean
def poll(timeout: Long, unit: TimeUnit)
def poll()
def take(): E
...
```

# DelayedItem
* 所在文件: core/src/main/scala/kafaka/utils/DelayedItem.scala
* 功能: 定义了可以放入到[DelayQueue](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/DelayQueue.html)队列的对象;
* 实现: 实现了Delayed接口;

# 先写这么多吧,其他的遇到的时候再来分析,不得不感叹java的类库真是丰富啊~~~