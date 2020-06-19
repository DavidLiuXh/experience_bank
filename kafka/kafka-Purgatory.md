* 在kafka中有很多操作需要延迟等待, 比如客户端发送数据到达leader后, 根据设置ack方式不同,需要等待其replicas返回ack, 那这个ack就需要延迟等待;对于一个拉取操作, 需要延迟等待期望拉取的字节数准备好;
* 有延迟操作, 那必然会存在操作的超时处理. 还记得我们上一篇[Kafka中的时间轮](http://www.jianshu.com/p/43132636166e)中对*Timer*的分析吧, 这里的延迟操作需要使用它来实现;
---
###### DelayedOperation
* 所在文件: core/src/main/scala/kafka/server/DelayedOperation.scala
* 这是个抽象类, 所有具体的延迟任务都需要继承这个类;
* 同时每个延迟任务必然存在操作的超时, 那么其超时操作是通过将对象放到[Kafka中的时间轮](http://www.jianshu.com/p/43132636166e)中的**Timer**中处理, 因此这个类又继承自`TimerTask`;
* ` private val completed = new AtomicBoolean(false) `: 原子变量, 标识当前operation是否已完成;
* `def forceComplete(): Boolean`: 强制完成操作;
```
if (completed.compareAndSet(false, true)) {
      // cancel the timeout timer
      cancel()
      onComplete()
      true
    } else {
      false
    }
```

 分两种情况:
 1. 当前操作已经完成,则不再需要强制完成,返回false;
 2. 当前操作未完成, 则首先在`Timer`中取消这个定时任务, 然后回调`onComplete`

* `override def run(): Unit`: 实现的是`TimerTask`的方法, 当超时时会执行此操作:
```
if (forceComplete())
      onExpiration()
```

 里面的操作比较简单, 调用`forceComplete`, 如果成功,表明是真的超时了,回调`onExpiration`;

* 需要由子类实现的方法:
 1. ` def onExpiration(): Unit`: 超时后的回调处理;
 2. `def onComplete(): Unit`: 操作完成后的回调处理;
 3. `def tryComplete(): Boolean`: 在放入到`Timer`前, 先尝试着执行一下这个操作, 看是否可以完成, 如果可以就不用放到`Timer`里了, 这是为了确保任务都尽快完成作的一个优化;

###### Watchers
* 所在文件: core/src/main/scala/kafka/server/DelayedOperation.scala
* 对于一个延迟任务, 一般会有两个操作加持在身:
 1. 上面说的作为超时任务放在`Timer`中;
 2. 与某些事件关联在一起, 可以关联多个事件, 当这些事件中的某一个发生时, 这个任务即可认为是完成;这个就是 `Watchers`类要完成的工作;
* `class Watchers(val key: Any)`: 构造时需要一个参数`key`,  你可以理解成是一个事件;
* `private[this] val operations = new LinkedList[T]()`: 用于存放和这个`key`关联的所有操作,一个`key`可以关联多个操作, 同时一个操作也可以被多个`key`关联(即位于多个`Watchers`对象中)
* `def purgeCompleted(): Int`: 删除链表中所有已经完成的操作
```
      var purged = 0
      operations synchronized {
        val iter = operations.iterator()
        while (iter.hasNext) {
          val curr = iter.next()
          if (curr.isCompleted) {
            iter.remove()
            purged += 1
          }
        }
      }
      if (operations.size == 0)
        removeKeyIfEmpty(key, this)

      purged
    }
```

* `def tryCompleteWatched(): Int `:
```
     var completed = 0
      operations synchronized {
        val iter = operations.iterator()
        while (iter.hasNext) {
          val curr = iter.next()
          if (curr.isCompleted) {
            // another thread has completed this operation, just remove it
            iter.remove()
          } else if (curr synchronized curr.tryComplete()) {
            completed += 1
            iter.remove()
          }
        }
      }

      if (operations.size == 0)
        removeKeyIfEmpty(key, this)

      completed
```

 扫描整个链表:
 1. 如果任务已完成,则移除;
 2. 如果任务未完成, 调用`tryComplete`尝试立即完成, 如果可以完成, 则移除;

* 添加任务: 
```
def watch(t: T) {
      operations synchronized operations.add(t)
    }
```

###### DelayedOperationPurgatory
* 所在文件: core/src/main/scala/kafka/server/DelayedOperation.scala
* 终于要揭开我们的**谜之炼狱**啦, 源码里的注释如下:
> A helper purgatory class for bookkeeping delayed operations with a timeout, and expiring timed out operations

实际上就是用来通过*Timer*和*Watchers*来管理一批延迟任务;

* `private[this] val timeoutTimer = new Timer(executor)`: 用来处理加入的作务的超时行为;
* `private val expirationReaper = new ExpiredOperationReaper()`: 
```
private class ExpiredOperationReaper extends ShutdownableThread(
    "ExpirationReaper-%d".format(brokerId),
    false) {

    override def doWork() {
      timeoutTimer.advanceClock(200L)

      if (estimatedTotalOperations.get - delayed > purgeInterval) {
        estimatedTotalOperations.getAndSet(delayed)
        debug("Begin purging watch lists")
        val purged = allWatchers.map(_.purgeCompleted()).sum
        debug("Purged %d elements from watch lists.".format(purged))
      }
    }
  }
```
 1. `timeoutTimer.advanceClock(200L)`: 驱动Timer向前走, pop出超时的延迟任务;
 2. `val purged = allWatchers.map(_.purgeCompleted()).sum`: 利用阈值(purgeInterval)来周期性地从`Watchers`中清理掉已经完成的任务;

* `def tryCompleteElseWatch(operation: T, watchKeys: Seq[Any]): Boolean`: 将operation和一系列的事件(key)关联起来, 然后调用`tryComplete`尝试立即完成该操作,如果不能完成,加入到Timer中;

* `def checkAndComplete(key: Any): Int`: 按key找到相应的Watchers对象, 然后调用其`tryCompleteWatched()`, 说明上面用;

###### 简图:

![DelayedOperation.png](http://upload-images.jianshu.io/upload_images/2020390-527b62fe8c8b49bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 基本上就是这些了
