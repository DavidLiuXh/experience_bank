#### Apache BookKeeper中数据目录分析
##### 需要落盘的数据
* Journals
 1. 这个journals文件里存储的相当于BookKeeper的事务log或者说是写前log, 在任何针对ledger的更新发生前，都会先将这个更新的描述信息持久化到这个journal文件中。
 2. Bookeeper提供有单独的sync线程根据当前journal文件的大小来作journal文件的rolling;

* EntryLogFile
 1. 存储真正数据的文件，写入的时候Entry数据先缓存在内存buffer中，然后批量flush到EntryLogFile中;
 2. 默认情况下，所有ledger的数据都是聚合然后顺序写入到同一个EntryLog文件中，避免磁盘随机写;

* Index文件
 1. 所有Ledger的entry数据都写入相同的EntryLog文件中，为了加速数据读取，会作 ledgerId + entryId 到文件offset的映射，这个映射会缓存在内存中，称为IndexCache;
 2. IndexCache容量达到上限时，会被 Sync线程flush到文件;

* LastLogMark
 1. 从上面的的讲述可知， 写入的EntryLog和Index都是先缓存在内存中，再根据一定的条件周期性的flush到磁盘，这就造成了从内存到持久化到磁盘的时间间隔，如果在这间隔内BookKeeper进程崩溃，在重启后，我们需要根据journal文件内容来恢复，这个`LastLogMark`就记录了从journal中什么位置开始恢复;
 2. 它其实是存在内存中，当IndexCache被flush到磁盘后其值会被更新，其也会周期性持久化到磁盘文件，供BookKeeper进程启动时读取来从journal中恢复;
 3. LastLogMark一旦被持久化到磁盘，即意味着在其之前的Index和EntryLog都已经被持久化到了磁盘，那么journal在这个LastLogMark之前的数据都可以被清除了。
 
###### 落盘数据目录设置优化 
* journal, entrylog, index最好设置在不同磁盘上，避免IO竞争;
* journal 最好写在SSD等高速磁盘上。

###### 数据写入后各种文件的更新流程
* 流程图

![data-flow1.png](https://upload-images.jianshu.io/upload_images/2020390-571827553b889a13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


###### 文件目录使用情况监控
* 用于写入文件的目录有三种状态：
  1. 可写;
  2. 可写，但剩余空间低于所配置的警告阈值;
  3. 不可写，已经写满; 当被GC清理了一部分数据后，其状态又可变为可写;

* BookKeeper需要持续监控目录空间使用情况, 通过 `LedgerDirsMonitor` 类实现, 我们主要来分析一下它的 `check` 方法， 注释写在函数体内
```
    private void check(final LedgerDirsManager ldm) {
	// 对于Index, EntryLog, Journal都可以设置多个存储路径, 每一种对应一个LedgerDirsManager
	// 先获取每种对应的dirs的使用情况
        final ConcurrentMap<File, Float> diskUsages = ldm.getDiskUsages();
        try {
		//获取当前可写状态的目录
            List<File> writableDirs = ldm.getWritableLedgerDirs();
            // Check all writable dirs disk space usage.
			// 循环遍历当前可写状态的dirs的剩余可写容量，更新diskUsages
			// 同时处理各种异常，比如
			// 1. 读dir失败，回调diskFailed
			// 2. 可写容量低于警戒阈值，但还处于可写状态， 回调 diskAlmostFull
			// 3. 不可写，调用ldm.addToFilledDirs
            for (File dir : writableDirs) {
                try {
                    diskUsages.put(dir, diskChecker.checkDir(dir));
                } catch (DiskErrorException e) {
                    LOG.error("Ledger directory {} failed on disk checking : ", dir, e);
                    // Notify disk failure to all listeners
                    for (LedgerDirsListener listener : ldm.getListeners()) {
                        listener.diskFailed(dir);
                    }
                } catch (DiskWarnThresholdException e) {
                    diskUsages.compute(dir, (d, prevUsage) -> {
                        if (null == prevUsage || e.getUsage() != prevUsage) {
                            LOG.warn("Ledger directory {} is almost full : usage {}", dir, e.getUsage());
                        }
                        return e.getUsage();
                    });
                    for (LedgerDirsListener listener : ldm.getListeners()) {
                        listener.diskAlmostFull(dir);
                    }
                } catch (DiskOutOfSpaceException e) {
                    diskUsages.compute(dir, (d, prevUsage) -> {
                        if (null == prevUsage || e.getUsage() != prevUsage) {
                            LOG.error("Ledger directory {} is out-of-space : usage {}", dir, e.getUsage());
                        }
                        return e.getUsage();
                    });
                    // Notify disk full to all listeners
                    ldm.addToFilledDirs(dir);
                }
            }
            // Let's get NoWritableLedgerDirException without waiting for the next iteration
            // in case we are out of writable dirs
            // otherwise for the duration of {interval} we end up in the state where
            // bookie cannot get writable dir but considered to be writable
			// check完之前所有可写目录的最新状态后，
			// 看看现在还有没有可写的目录，没有可用的就抛出异常
            ldm.getWritableLedgerDirs();
        } catch (NoWritableLedgerDirException e) {
            LOG.warn("LedgerDirsMonitor check process: All ledger directories are non writable");
            boolean highPriorityWritesAllowed = true;
            try {
                // disk check can be frequent, so disable 'loggingNoWritable' to avoid log flooding.
                ldm.getDirsAboveUsableThresholdSize(minUsableSizeForHighPriorityWrites, false);
            } catch (NoWritableLedgerDirException e1) {
                highPriorityWritesAllowed = false;
            }
			
			//进到这里，表明没有可写的目录了，回调allDisksFull
            for (LedgerDirsListener listener : ldm.getListeners()) {
                listener.allDisksFull(highPriorityWritesAllowed);
            }
        }

        List<File> fullfilledDirs = new ArrayList<File>(ldm.getFullFilledLedgerDirs());
        boolean makeWritable = ldm.hasWritableLedgerDirs();

        // When bookie is in READONLY mode, i.e there are no writableLedgerDirs:
        // - Update fullfilledDirs disk usage.
        // - If the total disk usage is below DiskLowWaterMarkUsageThreshold
        // add fullfilledDirs back to writableLedgerDirs list if their usage is < conf.getDiskUsageThreshold.
        try {
            if (!makeWritable) {
			    // 返回当前所有目录总的已用容量百分比
                float totalDiskUsage = diskChecker.getTotalDiskUsage(ldm.getAllLedgerDirs());
                if (totalDiskUsage < conf.getDiskLowWaterMarkUsageThreshold()) {
                    makeWritable = true;
                } else {
                    LOG.debug(
                        "Current TotalDiskUsage: {} is greater than LWMThreshold: {}."
                                + " So not adding any filledDir to WritableDirsList",
                        totalDiskUsage, conf.getDiskLowWaterMarkUsageThreshold());
                }
            }
            // Update all full-filled disk space usage
			// 之前处于不可写状态的目录，如果GC时清除掉一些数据，则可能变为可写状态，这里作check
            for (File dir : fullfilledDirs) {
                try {
                    diskUsages.put(dir, diskChecker.checkDir(dir));
                    if (makeWritable) {
                        ldm.addToWritableDirs(dir, true);
                    }
                } catch (DiskErrorException e) {
                    // Notify disk failure to all the listeners
                    for (LedgerDirsListener listener : ldm.getListeners()) {
                        listener.diskFailed(dir);
                    }
                } catch (DiskWarnThresholdException e) {
                    diskUsages.put(dir, e.getUsage());
                    // the full-filled dir become writable but still above the warn threshold
                    if (makeWritable) {
                        ldm.addToWritableDirs(dir, false);
                    }
                } catch (DiskOutOfSpaceException e) {
                    // the full-filled dir is still full-filled
                    diskUsages.put(dir, e.getUsage());
                }
            }
        } catch (IOException ioe) {
            LOG.error("Got IOException while monitoring Dirs", ioe);
            for (LedgerDirsListener listener : ldm.getListeners()) {
                listener.fatalError();
            }
        }
    }
```
