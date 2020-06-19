* 时间轮由来已久，Linux内核里有它，大大小小的应用里也用它;
* Kafka里主要用它来作大量的定时任务，超时判断等;
* 这里我们主要分析 Kafka中时间轮实现中用到的各个类.
***
###### TimerTask
*  所在文件：core/src/main/scala/kafka/utils/timer/TimerTask.scala
* 这个trait, 继承于 `Runnable`，需要放在时间轮里执行的任务都要继承这个`TimerTask`
* 每个`TimerTask`必须和一个`TimerTaskEntry`绑定，实现上放到时间轮里的是`TimerTaskEntry`
* `def cancel()`: 取消当前的Task, 实际是解除在当前`TaskEntry`上的绑定
```
def cancel(): Unit = {
    synchronized {
      if (timerTaskEntry != null) timerTaskEntry.remove()
      timerTaskEntry = null
    }
  }
```

###### TimerTaskEntry
* 所在文件：core/src/main/scala/kafka/utils/timer/TimerTaskList.scala
* 作用：绑定一个`TimerTask`对象，然后被加入到一个`TimerTaskLIst`中;
* 它是`TimerTaskList`这个***双向列表*** 中的元素，因此有如下三个成员：
```
  var list: TimerTaskList = null //属于哪一个TimerTaskList
  var next: TimerTaskEntry = null //指向其后一个元素 
  var prev: TimerTaskEntry = null //指向其前一个元素 
```
* `TimerTaskEntry`对象在构造成需要一个`TimerTask`对象，并且调用
```
timerTask.setTimerTaskEntry(this)
```
将`TimerTask`对象绑定到 `TimerTaskEntry`上
如果这个`TimerTask`对象之前已经绑定到了一个 `TimerTaskEntry`上, 先调用`timerTaskEntry.remove()`解除绑定。
* `def remove()`:
```
    var currentList = list
    while (currentList != null) {
      currentList.remove(this)
      currentList = list
    }
```
实际上就是把自己从当前所在`TimerTaskList`上摘掉, 为什么要使用一个`while(...)`来作，源码里是这样解释的：
> If remove is called when another thread is moving the entry from a task entry list to another, this may fail to remove the entry due to the change of value of list. 
Thus, we retry until the list becomes null.
In a rare case, this thread sees null and exits the loop, but the other thread insert the entry to another list later.

 简单说就是相当于用个自旋锁代替读写锁来尽量保证这个remove的操作的彻底。

###### TimerTaskList
* 所在文件：core/src/main/scala/kafka/utils/timer/TimerTaskList.scala
* 作为时间轮上的一个bucket, 是一个有头指针的双向链表
* 双向链表结构：
```
 private[this] val root = new TimerTaskEntry(null)
  root.next = root
  root.prev = root
```

* 继承于java的[Delayed](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Delayed.html)，说明这个对象应该是要被放入javar的[DelayQueue](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/DelayQueue.html)，自然要实现下面的两个接口：
```
def getDelay(unit: TimeUnit): Long = {
    unit.convert(max(getExpiration - SystemTime.milliseconds, 0), TimeUnit.MILLISECONDS)
  }

  def compareTo(d: Delayed): Int = {

    val other = d.asInstanceOf[TimerTaskList]

    if(getExpiration < other.getExpiration) -1
    else if(getExpiration > other.getExpiration) 1
    else 0
  }
```
* 每个 `TimerTaskList`都是时间轮上的一个bucket,自然也要关联一个过期
时间：`private[this] val expiration = new AtomicLong(-1L)`
* `add`和`remove`方法，用来添加和删除`TimerTaskEntry`
* `foreach`方法：在链表的每个元素上应用给定的函数;
* `flush`方法:在链表的每个元素上应用给定的函数,并清空整个链表, 同时超时时间也设置为-1;

###### TimingWheel
* 所在文件：core/src/main/scala/kafka/utils/timer/TimingWheel.scala
* 上面说了这么多，终于到这个时间轮出场了，说简单也简单，说复杂也复杂;
 1. 简言之，就是根据每个`TimerTaskEntry`的过期时间和当前时间轮的时间，选择一个合适的bucket(实际上就是`TimerTaskList`),这个桶的超时时间相同(会去余留整), 把这个`TimerTaskEntry`对象放进去，如果当前的`bucket`因超时被`DelayQueue`队列poll出来的话, 以为着这个`bucket`里面的都过期, 会调用这个`bucket`的`flush`方法, 将里面的entry都再次add一次,在这个add里因task已过期,将被立即提交执行,同时reset这个`bucket`的过期时间, 这样它就可以用来装入新的task了, **感谢我的同事"阔哥"的批评指正**.
 2. 这个时间轮是支持层级的，就是如果当前放入的`TimerTaskEntry`的过期时间如果超出了当前层级时间轮的覆盖范围，那么就创始一个`overflowWheel: TimingWheel`，放进去，只不过这个新的时间轮的降低了很多，那的tick是老时间轮的interval(相当于老时间轮的tick * wheelSize), 基本可以类比成钟表的分针和时针;
* `def add(timerTaskEntry: TimerTaskEntry): Boolean`: 将`TimerTaskEntry`加入适当的`TimerTaskList`;
* `def advanceClock(timeMs: Long):`:推动时间轮向前走，更新`CurrentTime`
* 值得注意的是，这个类***不是***线程安全的，也就是说`add`方法和`advanceClock`的调用方式使用者要来保证;
* 关于这个层级时间轮的原理，源码里有详细的说明.

###### Timer
* 所在文件：core/src/main/scala/kafka/utils/timer/Timer.scala
* 上面讲了这么多，现在是时候把这些组装起来了，这就是个用`TimingWheel`实现的定时器，可以添加任务，任务可以取消，可以到期被执行;
* 构造一个`TimingWheel`:
```
private[this] val delayQueue = new DelayQueue[TimerTaskList]()
  private[this] val taskCounter = new AtomicInteger(0)
  private[this] val timingWheel = new TimingWheel(
    tickMs = tickMs,
    wheelSize = wheelSize,
    startMs = startMs,
    taskCounter = taskCounter,
    delayQueue
  )
```

* `taskExecutor: ExecutorService`: 用于执行具体的task;
* 这个类为线程安全类，因为`TimingWhell`本身不是线程安全，所以对其操作需要加锁：
```
// Locks used to protect data structures while ticking
  private[this] val readWriteLock = new ReentrantReadWriteLock()
  private[this] val readLock = readWriteLock.readLock()
  private[this] val writeLock = readWriteLock.writeLock()
```

* `def add(timerTask: TimerTask):`:添加任务到定时器，通过调用`def addTimerTaskEntry(timerTaskEntry: TimerTaskEntry):`实现：
```
    if (!timingWheel.add(timerTaskEntry)) {
      // Already expired or cancelled
      if (!timerTaskEntry.cancelled)
        taskExecutor.submit(timerTaskEntry.timerTask)
    }
```
`timingWheel.add(timerTaskEntry)`:如果任务已经过期或被取消，则return false; 过期的任务被提交到`taskExcutor`执行;

* `def advanceClock(timeoutMs: Long)`:
```
    var bucket = delayQueue.poll(timeoutMs, TimeUnit.MILLISECONDS)
    if (bucket != null) {
      writeLock.lock()
      try {
        while (bucket != null) {
          timingWheel.advanceClock(bucket.getExpiration())
          bucket.flush(reinsert)
          bucket = delayQueue.poll()
        }
      } finally {
        writeLock.unlock()
      }
      true
    } else {
      false
    }
```
 1. `delayQueue.poll(timeoutMs, TimeUnit.MILLISECONDS)` 获取到期的bucket;
 2. 调用`timingWheel.advanceClock(bucket.getExpiration())`
 3. `bucket.flush(reinsert)`:对bucket中的每一个`TimerEntry`调用`reinsert`, 实际上是调用`addTimerTaskEntry(timerTaskEntry)`， 此时到期的Task会被执行;