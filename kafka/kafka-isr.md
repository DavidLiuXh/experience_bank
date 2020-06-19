* ISR列表: 所有同partiton leader数据同步的Replica集合;
* 在不允许partition leader脏选举的情况下, partition leader只能从ISR列表中选取;
* 根据ISR的定义可知, ISR列表的成员是有可能动态变化的, 集合可能被扩充, 也可能被收缩;
* ISR列表的维护由每个Partition的leader replica负责;
---
###### ISR列表收缩
- ReplicatManager在启动时会启动一个周期性任务, 来定期查看是否有ISR列表需要收缩: `scheduler.schedule("isr-expiration", maybeShrinkIsr, period = config.replicaLagTimeMaxMs, unit = TimeUnit.MILLISECONDS)`,  这个操作针对每个`partition`都进行检查, 最后会调用`Partition::maybeShrinkIsr`:
```
     val leaderHWIncremented = inWriteLock(leaderIsrUpdateLock) {
      leaderReplicaIfLocal() match {
        case Some(leaderReplica) =>
          val outOfSyncReplicas = getOutOfSyncReplicas(leaderReplica, replicaMaxLagTimeMs)
          if(outOfSyncReplicas.size > 0) {
            val newInSyncReplicas = inSyncReplicas -- outOfSyncReplicas
            assert(newInSyncReplicas.size > 0)
            info("Shrinking ISR for partition [%s,%d] from %s to %s".format(topic, partitionId,
              inSyncReplicas.map(_.brokerId).mkString(","), newInSyncReplicas.map(_.brokerId).mkString(",")))
            // update ISR in zk and in cache
            updateIsr(newInSyncReplicas)
            // we may need to increment high watermark since ISR could be down to 1

            replicaManager.isrShrinkRate.mark()
            maybeIncrementLeaderHW(leaderReplica) // ? 如果更新了HighWaterMark, 是否也要调用tryCompleteDelayedRequests()???
          } else {
            false
          }

        case None => false // do nothing if no longer leader
      }

```
  1. 核心是调用`getOutOfSyncReplicas`得到当前没有同步跟上leader的Replicat列表, 然后从`inSyncReplicas`中踢除掉后更新本地的metadata ISR缓存同时更新zk上`/brokers/topics/[topic]/partitions/[parition]/stat`的节点内容, 最后因为ISR列表成员减少了, 需要重新评估是否需要更新`leader`的`high water mark`;
  2. `getOutOfSyncReplicas`: 得到当前没有同步跟上leader的Replicat列表
```
    /**
     * there are two cases that will be handled here -
     * 1. Stuck followers: If the leo of the replica hasn't been updated for maxLagMs ms,
     *                     the follower is stuck and should be removed from the ISR
     * 2. Slow followers: If the replica has not read up to the leo within the last maxLagMs ms,
     *                    then the follower is lagging and should be removed from the ISR
     * Both these cases are handled by checking the lastCaughtUpTimeMs which represents
     * the last time when the replica was fully caught up. If either of the above conditions
     * is violated, that replica is considered to be out of sync
     *
     **/
    val leaderLogEndOffset = leaderReplica.logEndOffset
    val candidateReplicas = inSyncReplicas - leaderReplica

    val laggingReplicas = candidateReplicas.filter(r => (time.milliseconds - r.lastCaughtUpTimeMs) > maxLagMs)
    if(laggingReplicas.size > 0)
      debug("Lagging replicas for partition %s are %s".format(TopicAndPartition(topic, partitionId), laggingReplicas.map(_.brokerId).mkString(",")))

    laggingReplicas
```
 源码中的注释已经写得很清楚了.
  1. 被淘汰后ISR列表的条件是`(time.milliseconds - replicat.lastCaughtUpTimeMs) > maxLagMs`
  2. `replicat.lastCaughtUpTimeMs`何时被更新呢? 其实是 `Replica::updateLogResult`中:
```
   def updateLogReadResult(logReadResult : LogReadResult) {
    logEndOffset = logReadResult.info.fetchOffsetMetadata

    /* If the request read up to the log end offset snapshot when the read was initiated,
     * set the lastCaughtUpTimeMsUnderlying to the current time.
     * This means that the replica is fully caught up.
     */
    if(logReadResult.isReadFromLogEnd) {
      lastCaughtUpTimeMsUnderlying.set(time.milliseconds)
    }
  }
```
  3. 顺藤摸瓜,会发现在响应`FetchRequest`请求时即`ReplicaManager::fetchMessage`中的`updateFollowerLogReadResults(replicaId, logReadResults)`会调用 `Replica::updateLogResult`, 当处理当前的`FetchRequest`请求时,如果已经读取到了相应partiton leader的LogEndOffset了, 则可以更新`lastCaughtUpTimeMsUnderlying`, 表明当前的复本在这个`FetchRequest`请求返回后就进行同步跟上了`leader`的步伐;
  4. 有关响应`FetchRequest`请求的具体分析可参考[Kafka是如何处理客户端发送的数据的？](http://www.jianshu.com/p/561dcaff9a0b)
###### ISR列表扩容
- ISR扩容操作位于`Partition::maybeExpandIsr`中:
```
val leaderHWIncremented = inWriteLock(leaderIsrUpdateLock) {
      // check if this replica needs to be added to the ISR
      leaderReplicaIfLocal() match {
        case Some(leaderReplica) =>
          val replica = getReplica(replicaId).get
          val leaderHW = leaderReplica.highWatermark
          if(!inSyncReplicas.contains(replica) &&
             assignedReplicas.map(_.brokerId).contains(replicaId) &&
                  replica.logEndOffset.offsetDiff(leaderHW) >= 0) {
            val newInSyncReplicas = inSyncReplicas + replica
            info("Expanding ISR for partition [%s,%d] from %s to %s"
                         .format(topic, partitionId, inSyncReplicas.map(_.brokerId).mkString(","),
                                 newInSyncReplicas.map(_.brokerId).mkString(",")))
            // update ISR in ZK and cache
            updateIsr(newInSyncReplicas)
            replicaManager.isrExpandRate.mark()
          }

          // check if the HW of the partition can now be incremented
          // since the replica maybe now be in the ISR and its LEO has just incremented
          maybeIncrementLeaderHW(leaderReplica)

        case None => false // nothing to do if no longer leader
      }
    }

    // some delayed operations may be unblocked after HW changed
    if (leaderHWIncremented)
      tryCompleteDelayedRequests()
```
核心`replica.logEndOffset.offsetDiff(leaderHW) >= 0` 如果当前`replica`的`LEO`大于等于`Leader`的`HighWaterMark`, 则表明该`replica`的同步已经跟上了`leader`, 将其加入到ISR列表中,更新本地的metadata ISR缓存同时更新zk上`/brokers/topics/[topic]/partitions/[parition]/stat`的节点内容;
- `Partition::maybeExpandIsr`的调用时机: 在`Replica::updateReplicaLogReadResult`中被调用, 同样顺藤摸瓜,会发现也是在响应`FetchRequest`请求时即`ReplicaManager::fetchMessage`中的`updateFollowerLogReadResults(replicaId, logReadResults)`会调用;
###### ISR列表变化后, 更新集群内每台broker上的metadata
- 在上面的ISR列表收缩和扩容的同时,都会通过`ReplicaManager::recordIsrChange`来记录有变化的 `TopicAndParition`;
- `ReplicaManager`在启动时还会启动一个周期性任务`maybePropagateIsrChanges`, 来定期将ISR在变化的`TopicAndParition`信息写入zk的`/isr_change_notification`节点;
- `KafkaController`会监控zk的`/isr_change_notification`节点变化, 向所有的broker发送`MetadataRequest`;
- 我们来看看`maybePropagateIsrChanges`的实现:
```
  val now = System.currentTimeMillis()
    isrChangeSet synchronized {
      if (isrChangeSet.nonEmpty &&
        (lastIsrChangeMs.get() + ReplicaManager.IsrChangePropagationBlackOut < now ||
          lastIsrPropagationMs.get() + ReplicaManager.IsrChangePropagationInterval < now)) {
        ReplicationUtils.propagateIsrChanges(zkUtils, isrChangeSet)
        isrChangeSet.clear()
        lastIsrPropagationMs.set(now)
      }
    }
```
可以看到为了防止将频繁的ISR变化广播到整个集群, 这里作了限制;