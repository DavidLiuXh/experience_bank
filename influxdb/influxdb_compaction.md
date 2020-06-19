[toc]

## Influxdb中的Compaction操作

#### Compaction概述
* Influxdb的存储引擎使用了TSM文件结构，这其实也是在LSM-Tree基础针对时序特点作了改进，因此其与LSM-Tree类似，也有MemTable, WAL和SSTable;
* 既然是类似LSM-Tree，也需要Compation, 将内存MemTable的数据持久化到磁盘，将磁盘上的若干文件merge,以便减少文件个数，优化读效率;
* Influxdb的Compaction通常来说需要两步：
  1. 生成一个compaction计划，简单来说就是生成一组可以并行compaction的文件列表;
  2. 针对一组tsm文件来作compation;

#### Compaction计划的生成
##### CompactionPlanner接口
* 其实可以使用多种策略来生成这个计划，所谓的计划就是根据特定的规则来获取到一组可以并行作compact的文件组。因此Influxdb首先定义了一个Interface:
```
type CompactionPlanner interface {
	Plan(lastWrite time.Time) []CompactionGroup
	PlanLevel(level int) []CompactionGroup
	PlanOptimize() []CompactionGroup
	Release(group []CompactionGroup)
	FullyCompacted() bool

	// ForceFull causes the planner to return a full compaction plan the next
	// time Plan() is called if there are files that could be compacted.
	ForceFull()

	SetFileStore(fs *FileStore)
}
```

 `Plan`,`PlanLevel`,`PlanOptimize`返回的都是`[]CompactionGroup`, 它的类型其实是 `[][]string`, 即一组可以并行执行Compaction操作的tsm文件路径的列表;
 
##### CompactionPlanner的默认实现 - DefaultPlanner
* 在讲这个`DefaultPlanner`之前，我们先来看一下一个tsm文件的命名：`000001-01.tsm`，前面的`000001`被称为`Generation`, 后面的`01`被 称为`Sequence number`，也被称为`Level`

* `tsmGeneration`类型介绍： 它封装了属同一个Generation的多个TSM文件
```
type tsmGeneration struct {
	id            int // Generation
	files         []FileStat //包含的tsm文件的信息, 并且这个files是按文件名从小到大排序好的
	parseFileName ParseFileNameFunc //这个函数用来从tsm文件名中解析出Generation和Sequence number
}
```
因为在compact过程中针对同一个Generation，可以对应有多个不同的sequence,比如 `001-001.tsm, 001-002.tsm`, 这些都属于同一Generation, 下一次压缩时这两个文件可以被客视为一个大的`001-001.tsm`文件,这也就是需要这个tsmGeneration的原因

* `PlanLevel`: 针对某一level, 抽取出一组tsm文件组, 大概步骤为下：
 1. 根据当前file_store包含的所有tsm文件，将相同generation的文件归于一类，生成tsmGenerations, 这是通过`fileGenerations`完成的;
 2. 按level将上面得到的所有tsmGeneration分组, 最后得到的分组的成员是按tsmGenerations.Level()从大到小排列的
 3. 按PlanLevel(level int)中的level过滤上面得到的tsmGeneration group
 4. 将上面得到的每个tsmGeneration group中的tsmGeneratons按指定大小分堆，作chunk, 这些分为的堆中的tsm文件按堆可以被并行compact；
 5. 代码有点多，也不太直观，大体上是这个思路;
```
func (c *DefaultPlanner) PlanLevel(level int) []CompactionGroup {
    ...	
	
	// Determine the generations from all files on disk.  We need to treat
	// a generation conceptually as a single file even though it may be
	// split across several files in sequence.
	// 将相同generation id的tsm文件放在一起
	// generations -> tsmGenerations
	generations := c.findGenerations(true)
    ...
	
	// 按level把tsmGenerations分组
	// 这些分完组后groups中的tsmGenerations的level从大到小排列的
	var currentGen tsmGenerations
	var groups []tsmGenerations
	for i := 0; i < len(generations); i++ {
		cur := generations[i]

		// See if this generation is orphan'd which would prevent it from being further
		// compacted until a final full compactin runs.
		if i < len(generations)-1 {
			if cur.level() < generations[i+1].level() {
				currentGen = append(currentGen, cur)
				continue
			}
		}

		if len(currentGen) == 0 || currentGen.level() == cur.level() {
			currentGen = append(currentGen, cur)
			continue
		}
		groups = append(groups, currentGen)
		currentGen = tsmGenerations{}
		currentGen = append(currentGen, cur)
	}
	if len(currentGen) > 0 {
		groups = append(groups, currentGen)
	}

	// Remove any groups in the wrong level
	// level是这个函数传进来的参数，指明要compact哪一level的file,这里作个过滤
	// cur.level()返回的是这个tmsGeneration中所有fileState中最小的level, 这样作
	// 合适吗？
	var levelGroups []tsmGenerations
	for _, cur := range groups {
		if cur.level() == level {
			levelGroups = append(levelGroups, cur)
		}
	}

	minGenerations := 4
	if level == 1 {
		//对于level至少要有8个文件，才会compact
		minGenerations = 8
	}

	//type CompactionGroup []string
	var cGroups []CompactionGroup
	for _, group := range levelGroups {
		// 将每个tsmGenerations中的tsmGeneration按给定大小分堆
		for _, chunk := range group.chunk(minGenerations) {
			var cGroup CompactionGroup
			var hasTombstones bool
			for _, gen := range chunk {
				if gen.hasTombstones() {
					hasTombstones = true
				}
				for _, file := range gen.files {
					//cGroup里存需要被分组compact的file.Path
					cGroup = append(cGroup, file.Path)
				}
			}

			// 如果当前的chunk里的tsmGeneration数不够minGeneration大小，
			// 需要用下一个chunk来凑够这个数
			// hasTombstones为true, 说明有标记删除的，需要通过 compact 真正删除掉
			if len(chunk) < minGenerations && !hasTombstones {
				continue
			}

			cGroups = append(cGroups, cGroup)
		}
	}

	if !c.acquire(cGroups) {
		return nil
	}
	return cGroups
}
```
* `Plan(lastWrite time.Time)`: 针对full compaction或level >= 4的generation产生一组tsm文件组
 1. 代码可以说是又臭又长，规则读起来说实话也不是完全明白;
 2. fullCompaction是有时间间隔的，满足了这个时间间隔，作fullCompaction；而且需要根据一些条件作排除;
 3. 如果不作fullCompaction, 那就只针对generation.level >= 4的 generations生成compaction计划;
 4. 我把代码放在下面，里面有一些注释：
```
func (c *DefaultPlanner) Plan(lastWrite time.Time) []CompactionGroup {
	generations := c.findGenerations(true)

	for _, v := range generations {
		fmt.Printf("xxx | generations: %v\n", v)
	}

	c.mu.RLock()
	forceFull := c.forceFull
	c.mu.RUnlock()

	// first check if we should be doing a full compaction because nothing has been written in a long time
	// fullCompact是有时间间隔的，这里作判断
	// 这部分处理fullCompact的情况
	if forceFull || c.compactFullWriteColdDuration > 0 && time.Since(lastWrite) > c.compactFullWriteColdDuration && len(generations) > 1 {

		// Reset the full schedule if we planned because of it.
		if forceFull {
			c.mu.Lock()
			c.forceFull = false
			c.mu.Unlock()
		}

		var tsmFiles []string
		var genCount int
		for i, group := range generations {
			var skip bool

			// Skip the file if it's over the max size and contains a full block and it does not have any tombstones
			if len(generations) > 2 && group.size() > uint64(maxTSMFileSize) &&
				c.FileStore.BlockCount(group.files[0].Path, 1) == tsdb.DefaultMaxPointsPerBlock &&
				!group.hasTombstones() {
				skip = true
			}

			// compressed files.
			if i < len(generations)-1 {
				if generations[i+1].level() <= 3 {
					skip = false
				}
			}

			if skip {
				continue
			}

			for _, f := range group.files {
				tsmFiles = append(tsmFiles, f.Path)
			}
			genCount += 1
		}
		sort.Strings(tsmFiles)

		// Make sure we have more than 1 file and more than 1 generation
		if len(tsmFiles) <= 1 || genCount <= 1 {
			return nil
		}

		group := []CompactionGroup{tsmFiles}
		if !c.acquire(group) {
			return nil
		}
		return group
	}

	// don't plan if nothing has changed in the filestore
	if c.lastPlanCheck.After(c.FileStore.LastModified()) && !generations.hasTombstones() {
		return nil
	}

	c.lastPlanCheck = time.Now()

	// If there is only one generation, return early to avoid re-compacting the same file
	// over and over again.
	if len(generations) <= 1 && !generations.hasTombstones() {
		return nil
	}

	// Need to find the ending point for level 4 files.  They will be the oldest files. We scan
	// each generation in descending break once we see a file less than 4.
	end := 0
	start := 0
	// 因为finGeneratons返回的是按level从大到小排序的
	// 这里找到level >= 4的截至点
	for i, g := range generations {
		if g.level() <= 3 {
			break
		}
		end = i + 1
	}

	// As compactions run, the oldest files get bigger.  We don't want to re-compact them during
	// this planning if they are maxed out so skip over any we see.
	var hasTombstones bool
	for i, g := range generations[:end] {
		if g.hasTombstones() {
			hasTombstones = true
		}

		if hasTombstones {
			continue
		}

        // 下面这部分主要是跳到过大的tsm文件
		// Skip the file if it's over the max size and contains a full block or the generation is split
		// over multiple files.  In the latter case, that would mean the data in the file spilled over
		// the 2GB limit.
		if g.size() > uint64(maxTSMFileSize) &&
			c.FileStore.BlockCount(g.files[0].Path, 1) == tsdb.DefaultMaxPointsPerBlock {
			start = i + 1
		}

		// This is an edge case that can happen after multiple compactions run.  The files at the beginning
		// can become larger faster than ones after them.  We want to skip those really big ones and just
		// compact the smaller ones until they are closer in size.
		if i > 0 {
			if g.size()*2 < generations[i-1].size() {
				start = i
				break
			}
		}
	}

	// step is how may files to compact in a group.  We want to clamp it at 4 but also stil
	// return groups smaller than 4.
	step := 4
	if step > end {
		step = end
	}

	// slice off the generations that we'll examine
	generations = generations[start:end]

    // 下面这些代码主要就是将generations分堆，也就是最后要将tsm文件分堆，以便并行作compaction
	// Loop through the generations in groups of size step and see if we can compact all (or
	// some of them as group)
	groups := []tsmGenerations{}
	for i := 0; i < len(generations); i += step {
		var skipGroup bool
		startIndex := i

		for j := i; j < i+step && j < len(generations); j++ {
			gen := generations[j]
			lvl := gen.level()

			// Skip compacting this group if there happens to be any lower level files in the
			// middle.  These will get picked up by the level compactors.
			if lvl <= 3 {
				fmt.Printf("xxx | lvl <= 3")
				skipGroup = true
				break
			}

			// Skip the file if it's over the max size and it contains a full block
			if gen.size() >= uint64(maxTSMFileSize) && c.FileStore.BlockCount(gen.files[0].Path, 1) == tsdb.DefaultMaxPointsPerBlock && !gen.hasTombstones() {
				startIndex++
				continue
			}
		}

		if skipGroup {
			continue
		}

		endIndex := i + step
		if endIndex > len(generations) {
			endIndex = len(generations)
		}
		if endIndex-startIndex > 0 {
			groups = append(groups, generations[startIndex:endIndex])
		}
	}

	if len(groups) == 0 {
		return nil
	}

	// With the groups, we need to evaluate whether the group as a whole can be compacted
	compactable := []tsmGenerations{}
	for _, group := range groups {
		//if we don't have enough generations to compact, skip it
		if len(group) < 4 && !group.hasTombstones() {
			continue
		}
		compactable = append(compactable, group)
	}

	// All the files to be compacted must be compacted in order.  We need to convert each
	// group to the actual set of files in that group to be compacted.
	var tsmFiles []CompactionGroup
	for _, c := range compactable {
		var cGroup CompactionGroup
		for _, group := range c {
			for _, f := range group.files {
				cGroup = append(cGroup, f.Path)
			}
		}
		sort.Strings(cGroup)
		tsmFiles = append(tsmFiles, cGroup)
	}

	if !c.acquire(tsmFiles) {
		return nil
	}
	return tsmFiles
}
```

* 针对这些compaction策略，我将一般情况用张图表明一下，它不能涵盖所有情况，只作为一般性参考:
![compaction stragy](file:///home/lw/downloads/5206a8b540ac4adc8a69d980bb9fb523.jpg)

#### Compation的执行
##### Compactor-Compaction的执行者
两个作用：
* 将内存的Cache(MemTable)持久化到磁盘TSM文件(SSTable), Influxdb中叫`写快照`
* 将磁盘上的多个TSM文件作merge

##### 持久化Cache到TSM文件
###### Cache回顾
* 先回顾一下Cache的构成，简单说就是个Key-Value，为了降低读写时锁的竞争，又引入了partiton(桶)的概念，每个partition里又是一个key-value的map;Key通过hash选择一个partition
* 这里的key是series key + filed, value就是具体的存入influxdb的用户数据
![compaction stragy](file:///home/lw/downloads/618da1d984c8d48961950ab9bd681b31.jpg)
* 持久化就是将这些key-value存到磁盘，在存之前还要作encode;
* 按influxdb代码的一贯写法，这里在写入磁盘时需要一个iterator来遍历所有的key-value

###### Cache的遍历
* 上面的这些功能都通过`cacheKeyIterator`完成， 它提供了按key遍历的功能，并且在遍历前已经对Values（包含value和时间戳）作了列编码;
  1. 这个编译过程会启动多个goroutine并行进行
  2. 针对Cache中的每个key对应的values，都单独编码，结果记录在`c.blocks`中，Caceh中有几个key，c.blocks中就有几项
  3. 对于同一个key的所有values,也不是统一编码到一块block中，每一个cacheBlock最多容纳`c.size`个vlaues
```
func (c *cacheKeyIterator) encode() {
	concurrency := runtime.GOMAXPROCS(0)
	n := len(c.ready)

	// Divide the keyset across each CPU
	chunkSize := 1
	idx := uint64(0)

    // 启动多个goroutine来作encode
	for i := 0; i < concurrency; i++ {
		// Run one goroutine per CPU and encode a section of the key space concurrently
		go func() {
		    // 获取Time, Float, Boolean, Unsigned, String, Iterger的编码器
			tenc := getTimeEncoder(tsdb.DefaultMaxPointsPerBlock)
			fenc := getFloatEncoder(tsdb.DefaultMaxPointsPerBlock)
			benc := getBooleanEncoder(tsdb.DefaultMaxPointsPerBlock)
			uenc := getUnsignedEncoder(tsdb.DefaultMaxPointsPerBlock)
			senc := getStringEncoder(tsdb.DefaultMaxPointsPerBlock)
			ienc := getIntegerEncoder(tsdb.DefaultMaxPointsPerBlock)

			defer putTimeEncoder(tenc)
			defer putFloatEncoder(fenc)
			defer putBooleanEncoder(benc)
			defer putUnsignedEncoder(uenc)
			defer putStringEncoder(senc)
			defer putIntegerEncoder(ienc)

			for {
				i := int(atomic.AddUint64(&idx, uint64(chunkSize))) - chunkSize

				if i >= n {
					break
				}

				key := c.order[i]
				values := c.cache.values(key)

				for len(values) > 0 {

                    //每次最多编码c.size个value
					end := len(values)
					if end > c.size {
						end = c.size
					}

					minTime, maxTime := values[0].UnixNano(), values[end-1].UnixNano()
					var b []byte
					var err error

					switch values[0].(type) {
					case FloatValue:
						b, err = encodeFloatBlockUsing(nil, values[:end], tenc, fenc)
					case IntegerValue:
						b, err = encodeIntegerBlockUsing(nil, values[:end], tenc, ienc)
					case UnsignedValue:
						b, err = encodeUnsignedBlockUsing(nil, values[:end], tenc, uenc)
					case BooleanValue:
						b, err = encodeBooleanBlockUsing(nil, values[:end], tenc, benc)
					case StringValue:
						b, err = encodeStringBlockUsing(nil, values[:end], tenc, senc)
					default:
						b, err = Values(values[:end]).Encode(nil)
					}

                    // 更新values为剩余未编码的
					values = values[end:]

                    // 每个key对应c.blocks中的一项，里面存储的是cacheBlock
					c.blocks[i] = append(c.blocks[i], cacheBlock{
						k:       key,
						minTime: minTime,
						maxTime: maxTime,
						b:       b,
						err:     err,
					})

					if err != nil {
						c.err = err
					}
				}
				
				// Notify this key is fully encoded
				// 对于每个key, 如果全部编码完成，就向这个key对应的chan中写入数据，通知其编码完成
				c.ready[i] <- struct{}{}
			}
		}()
	}
}
```

* 编码结果的遍历
 1. `Next()`:
 ```
 func (c *cacheKeyIterator) Next() bool {
	//c.i的初值是 -1, 第一次调用或当前c.blocks[c.i]中已读取完，则下面的if不会进入
	if c.i >= 0 && c.i < len(c.ready) && len(c.blocks[c.i]) > 0 {
		c.blocks[c.i] = c.blocks[c.i][1:]
		if len(c.blocks[c.i]) > 0 {
			return true
		}
	}
	c.i++

	if c.i >= len(c.ready) {
		return false
	}

    // 这里阻塞等待对应的key编码完成
	<-c.ready[c.i]
	return true
}
 ```
 2. `read`读取:
 ```
 func (c *cacheKeyIterator) Read() ([]byte, int64, int64, []byte, error) {
	// See if snapshot compactions were disabled while we were running.
	select {
	case <-c.interrupt:
		c.err = errCompactionAborted{}
		return nil, 0, 0, nil, c.err
	default:
	}

	blk := c.blocks[c.i][0]
	return blk.k, blk.minTime, blk.maxTime, blk.b, blk.err
}
 ```
 
* Cache的Compaction操作：
  1. 先根据cache的规模和cache产生的速度确定是否需要作流控和compact的并发度
  2. 根据并发度将Cache分裂成若干个小规模Cache，每个小Cache对应一个goroutine来作compaction
  3. compaction过程是通过遍历相应的cacheKeyIterator来写入文件`c.writeNewFiles`
  4. 对于每个并发执行的`c.writeNewFiles`, 都对应不同的Generation, Sequence number都从0开始
  5. 
```
// 将Cache的内容写入到 *.tsm.tmp文件中
// cache中value过多的话，会将cache作split成多个cache,并行处理，每个splited cache有自己的generation
func (c *Compactor) WriteSnapshot(cache *Cache) ([]string, error) {
	c.mu.RLock()
	enabled := c.snapshotsEnabled
	intC := c.snapshotsInterrupt
	c.mu.RUnlock()

	if !enabled {
		return nil, errSnapshotsDisabled
	}

	start := time.Now()
	// cache.Count() 返回cache的所有的 value的个数
	card := cache.Count()

	// Enable throttling if we have lower cardinality or snapshots are going fast.
	// 3e6 = 3 x 10的6次方
    // compaction过程是否要作流控
	throttle := card < 3e6 && c.snapshotLatencies.avg() < 15*time.Second

	// Write snapshost concurrently if cardinality is relatively high.
	concurrency := card / 2e6
	if concurrency < 1 {
		concurrency = 1
	}

	// Special case very high cardinality, use max concurrency and don't throttle writes.
	if card >= 3e6 {
		concurrency = 4
		throttle = false
	}

	splits := cache.Split(concurrency)

	type res struct {
		files []string
		err   error
	}

	resC := make(chan res, concurrency)
	for i := 0; i < concurrency; i++ {
		go func(sp *Cache) {
			iter := NewCacheKeyIterator(sp, tsdb.DefaultMaxPointsPerBlock, intC)
			files, err := c.writeNewFiles(c.FileStore.NextGeneration(), 0, nil, iter, throttle)
			resC <- res{files: files, err: err}

		}(splits[i])
	}

	var err error
	files := make([]string, 0, concurrency)
	for i := 0; i < concurrency; i++ {
		result := <-resC
		if result.err != nil {
			err = result.err
		}
		files = append(files, result.files...)
	}

    ...	

	return files, err
}
```

* 遍历`keyIterator`,将编码后的block写入到tsm文件 `writeNewFiles`
  1. 主要就是调用tsmWriter的方法写入文件
  2. 写入文件时先写具体的block, 再写索引 
  3. 文件的大小或block数达到上限时，切下一个文件
  4. 
```
func (c *Compactor) writeNewFiles(generation, sequence int, src []string, iter KeyIterator, throttle bool) ([]string, error) {
	// These are the new TSM files written
	var files []string

	for {
	    // sequence + 1, 这个sequence其实就是 level
		sequence++

		// 这里写入的文件的命名为 *.tsm.tmp
		// 它在作fullCompact时被重命名为 *.tsm
		fileName := filepath.Join(c.Dir, c.formatFileName(generation, sequence)+"."+TSMFileExtension+"."+TmpTSMFileExtension)

		// Write as much as possible to this file
		// c.write实现了实际的写入操作
		err := c.write(fileName, iter, throttle)

		// We've hit the max file limit and there is more to write.  Create a new file
		// and continue.
		// 写入的文件大小或block数达到上限，就切下一个文件，sequence + 1
		if err == errMaxFileExceeded || err == ErrMaxBlocksExceeded {
			files = append(files, fileName)
			continue
		} else if err == ErrNoValues {
		    // ErrNoValues意味着没有有效的value, 只有tombstoned entires, 就不写入文件
			// If the file only contained tombstoned entries, then it would be a 0 length
			// file that we can drop.
			if err := os.RemoveAll(fileName); err != nil {
				return nil, err
			}
			break
		} else if _, ok := err.(errCompactionInProgress); ok {
			// Don't clean up the file as another compaction is using it.  This should not happen as the
			// planner keeps track of which files are assigned to compaction plans now.
			return nil, err
		} else if err != nil {
			// Remove any tmp files we already completed
			for _, f := range files {
				if err := os.RemoveAll(f); err != nil {
					return nil, err
				}
			}
			// We hit an error and didn't finish the compaction.  Remove the temp file and abort.
			if err := os.RemoveAll(fileName); err != nil {
				return nil, err
			}
			return nil, err
		}

		files = append(files, fileName)
		break
	}

	return files, nil
}
```

##### 多个tsm文件的compaction
###### 概述
我们先来简单讲一下这个compaction的过程，这类似于归并合并操作，每个tsm文件中的keys在其索引中都是从小到小排序的，compaction时就是将多个文件中的相同key的block合并在一起，再生成新的索引，说起来就是这么简单，但influxdb在实现时为了效率等作了一些额外的策略;
###### tsmBatchKeyIterator
* 和上面的Cache的compatcon一样，这里也需要一个Iterator： `tsmBatchKeyIterator`， 它用来同时遍历多个tsm文件, 这个是compaction过程的精华所在

###### tsmBatchKeyIterator的遍历
 1. 先将各tsm文件中的第一个key对应的block一一取出
 2. 扫描1中获取到的所有每一个key,确定一个当前最小的key
 3. 从1中获取到的所有block中提取出key等于2中获取的最小key的block,存在`k.blocks`中
 4. 对3中获取的所有block作merge, 主要是按minTime排序，这样基本就完成了一个Next的操作
 5. 具体代码如下，我在里面加了注释
 6. 
```
 func (k *tsmBatchKeyIterator) Next() bool {
RETRY:
	// Any merged blocks pending?
	if len(k.merged) > 0 {
		k.merged = k.merged[1:]
		if len(k.merged) > 0 {
			return true
		}
	}

	// Any merged values pending?
	if k.hasMergedValues() {
		k.merge()
		if len(k.merged) > 0 || k.hasMergedValues() {
			return true
		}
	}

	// If we still have blocks from the last read, merge them
	if len(k.blocks) > 0 {
		k.merge()
		if len(k.merged) > 0 || k.hasMergedValues() {
			return true
		}
	}

	// Read the next block from each TSM iterator
	// 读每一个tsm文件，将其第一组block都存到k.buf里，看起来是要合并排序
	// 每个tsm文件对应一个blocks
	// 这个blocks和tsm的index是一样的，是按key从小到大排序的
	for i, v := range k.buf {
		if len(v) != 0 {
			continue
		}

		iter := k.iterators[i]
		if iter.Next() {
			key, minTime, maxTime, typ, _, b, err := iter.Read()
			if err != nil {
				k.err = err
			}

			// This block may have ranges of time removed from it that would
			// reduce the block min and max time.
			// 这个tombstones是[]TimeRange
			tombstones := iter.r.TombstoneRange(key)

			var blk *block
			// k.buf[i]的类型是[]blocks -> [][]block
			// 下面这段逻辑，就是不断向k.buf[i]中append新的bolck
			// 如果k.buf[i]需要扩容，就在append时扩，扩为原有cap的二倍
			if cap(k.buf[i]) > len(k.buf[i]) {
				k.buf[i] = k.buf[i][:len(k.buf[i])+1]
				blk = k.buf[i][len(k.buf[i])-1]
				if blk == nil {
					blk = &block{}
					k.buf[i][len(k.buf[i])-1] = blk
				}
			} else {
				blk = &block{}
				k.buf[i] = append(k.buf[i], blk)
			}

			blk.minTime = minTime
			blk.maxTime = maxTime
			blk.key = key
			blk.typ = typ
			blk.b = b
			blk.tombstones = tombstones
			blk.readMin = math.MaxInt64
			blk.readMax = math.MinInt64

			blockKey := key
			// 如果这两个key相等，说明还没有遍历完当前的block
			for bytes.Equal(iter.PeekNext(), blockKey) {
				iter.Next()
				key, minTime, maxTime, typ, _, b, err := iter.Read()
				if err != nil {
					k.err = err
				}

				tombstones := iter.r.TombstoneRange(key)

				var blk *block
				if cap(k.buf[i]) > len(k.buf[i]) {
					k.buf[i] = k.buf[i][:len(k.buf[i])+1]
					blk = k.buf[i][len(k.buf[i])-1]
					if blk == nil {
						blk = &block{}
						k.buf[i][len(k.buf[i])-1] = blk
					}
				} else {
					blk = &block{}
					k.buf[i] = append(k.buf[i], blk)
				}

				blk.minTime = minTime
				blk.maxTime = maxTime
				blk.key = key
				blk.typ = typ
				blk.b = b
				blk.tombstones = tombstones
				blk.readMin = math.MaxInt64
				blk.readMax = math.MinInt64
			}
		}

		if iter.Err() != nil {
			k.err = iter.Err()
		}
	}

	// Each reader could have a different key that it's currently at, need to find
	// the next smallest one to keep the sort ordering.
	// 找出当前最小的key(series key + field)
	// 因为k.buf中的每个blocks都是按key从小到大排好的，
	// 所以这里只需看每个blocks[0]
	var minKey []byte
	var minType byte
	for _, b := range k.buf {
		// block could be nil if the iterator has been exhausted for that file
		if len(b) == 0 {
			continue
		}
		if len(minKey) == 0 || bytes.Compare(b[0].key, minKey) < 0 {
			minKey = b[0].key
			minType = b[0].typ
		}
	}
	k.key = minKey
	k.typ = minType

	// Now we need to find all blocks that match the min key so we can combine and dedupe
	// the blocks if necessary
	// 把key都等于上面获取的minKey的block放到k.blocks中
	for i, b := range k.buf {
		if len(b) == 0 {
			continue
		}
		//b[0]即为当前的k.buf[i][0], 是一个block
		// b是[]block
		if bytes.Equal(b[0].key, k.key) {
			//k.blocks => []block
			// b => []block
			k.blocks = append(k.blocks, b...)
			//k.buf[i]的length被reset为0, 即已有的数据被清掉
			k.buf[i] = k.buf[i][:0]
		}
	}

	if len(k.blocks) == 0 {
		return false
	}

	k.merge()

	// After merging all the values for this key, we might not have any.  (e.g. they were all deleted
	// through many tombstones).  In this case, move on to the next key instead of ending iteration.
	if len(k.merged) == 0 {
		goto RETRY
	}

	return len(k.merged) > 0
}
```

###### tsmBtchKeyIterator的合并
 1. 当前需要合并的block都存在`k.blocks`里，先将其按`block.minTime`排序;
 2. 判断是否需要去重，如果`k.blocks`中的block在[minTime, maxTime]上有重叠或者某个block有tombstones，就都需要重构这些block,需要作去重,删除，重排操作, 相当于将所有的block按minTime重新组合排序;
 3. 我们来看下关键代码，里面我添加了一些注释
 4. 
```
func (k *tsmBatchKeyIterator) combineFloat(dedup bool) blocks {
	if dedup {
		//实现了按minTime来排序,去重
		for k.mergedFloatValues.Len() < k.size && len(k.blocks) > 0 {
			// 去除已经读取过的block
			for len(k.blocks) > 0 && k.blocks[0].read() {
				k.blocks = k.blocks[1:]
			}

			if len(k.blocks) == 0 {
				break
			}
			first := k.blocks[0]
			minTime := first.minTime
			maxTime := first.maxTime

			// Adjust the min time to the start of any overlapping blocks.
			// 其实i可以从1开始
			// 为了按minTime排序，需要确定一个全局最小范围的[minTime, maxTime]
			for i := 0; i < len(k.blocks); i++ {
				if k.blocks[i].overlapsTimeRange(minTime, maxTime) && !k.blocks[i].read() {
					if k.blocks[i].minTime < minTime {
						minTime = k.blocks[i].minTime
					}

					// 将最大值减小
					if k.blocks[i].maxTime > minTime && k.blocks[i].maxTime < maxTime {
						maxTime = k.blocks[i].maxTime
					}
				}
			}

			// We have some overlapping blocks so decode all, append in order and then dedup
			// 按上面确定的[minTime, maxTime]在所有的blocks中捞数据
			for i := 0; i < len(k.blocks); i++ {
				if !k.blocks[i].overlapsTimeRange(minTime, maxTime) || k.blocks[i].read() {
					continue
				}

				var v tsdb.FloatArray
				var err error
				if err = DecodeFloatArrayBlock(k.blocks[i].b, &v); err != nil {
					k.err = err
					return nil
				}

				// Remove values we already read
				v.Exclude(k.blocks[i].readMin, k.blocks[i].readMax)

				// Filter out only the values for overlapping block
				// 这个Include是不是可以不用调用
				v.Include(minTime, maxTime)
				if v.Len() > 0 {
					// Record that we read a subset of the block
					k.blocks[i].markRead(v.MinTime(), v.MaxTime())
				}

				// Apply each tombstone to the block
				for _, ts := range k.blocks[i].tombstones {
					v.Exclude(ts.Min, ts.Max)
				}

				k.mergedFloatValues.Merge(&v)
			}
		}

		// Since we combined multiple blocks, we could have more values than we should put into
		// a single block.  We need to chunk them up into groups and re-encode them.
		return k.chunkFloat(nil)
	}
	var i int

	for i < len(k.blocks) {

		// skip this block if it's values were already read
		if k.blocks[i].read() {
			i++
			continue
		}
		// If we this block is already full, just add it as is
		// 遇到一个不full的Block就break, 那如果后续还有full的block怎么办?
		if BlockCount(k.blocks[i].b) >= k.size {
			k.merged = append(k.merged, k.blocks[i])
		} else {
			break
		}
		i++
	}

	if k.fast {
		for i < len(k.blocks) {
			// skip this block if it's values were already read
			if k.blocks[i].read() {
				i++
				continue
			}

			k.merged = append(k.merged, k.blocks[i])
			i++
		}
	}

	// If we only have 1 blocks left, just append it as is and avoid decoding/recoding
	if i == len(k.blocks)-1 {
		if !k.blocks[i].read() {
			k.merged = append(k.merged, k.blocks[i])
		}
		i++
	}

	// The remaining blocks can be combined and we know that they do not overlap and
	// so we can just append each, sort and re-encode.
	for i < len(k.blocks) && k.mergedFloatValues.Len() < k.size {
		if k.blocks[i].read() {
			i++
			continue
		}

		var v tsdb.FloatArray
		if err := DecodeFloatArrayBlock(k.blocks[i].b, &v); err != nil {
			k.err = err
			return nil
		}

		// Apply each tombstone to the block
		for _, ts := range k.blocks[i].tombstones {
			v.Exclude(ts.Min, ts.Max)
		}

		k.blocks[i].markRead(k.blocks[i].minTime, k.blocks[i].maxTime)

		k.mergedFloatValues.Merge(&v)
		i++
	}

	k.blocks = k.blocks[i:]

	return k.chunkFloat(k.merged)
}
```

