[toc]

## TSI索引解析之TSL文件
* InfluxDB官网文档是很多的参考
[In-memory indexing and the Time-Structured Merge Tree (TSM)](https://docs.influxdata.com/influxdb/v1.7/concepts/storage_engine/)
[ime Series Index (TSI) overview](https://docs.influxdata.com/influxdb/v1.7/concepts/time-series-index/)
[Time Series Index (TSI) details](https://docs.influxdata.com/influxdb/v1.7/concepts/tsi-details)

#### LogFile解析
* 作用： LogFile操作新写入的Series在内存中的索引和持久化到WAL的;
* 下面我们先来看一下用到的相关数据结构

###### logTagValue
* 定义：代表一个TagKey对应的一个TagValue
```
type logTagValue struct {
	name      []byte  // tag value的值 
	deleted   bool //是否已经被删除
	series    map[uint64]struct{} // 属于哪些series id
	seriesSet *tsdb.SeriesIDSet
}
```
`series`和`seriesSet`其实都是用来存储SeriesID, 当SeriesID数量小于等于25个时，存到`series`里，反之存到`seriesSet`这个roaring bitmap里;
* 添加SeriesID:
```
func (tv *logTagValue) addSeriesID(x uint64) {
	if tv.seriesSet != nil {
		tv.seriesSet.AddNoLock(x)
		return
	}

    //数据量小就存在series map里
	tv.series[x] = struct{}{}

    //数据量大存在series这个roaring bitmap里
	if len(tv.series) > 25 {
		tv.seriesSet = tsdb.NewSeriesIDSet()
		for id := range tv.series {
			tv.seriesSet.AddNoLock(id)
		}
		tv.series = nil
	}
}
```
* 删除SeriesID: `removeSeriesID`
* 获取SeriesID的基数，所谓的基数就是不相同的SeriesID的个数
```
func (tv *logTagValue) cardinality() int64 {
	if tv.seriesSet != nil {
		return int64(tv.seriesSet.Cardinality())
	}
	return int64(len(tv.series))
}
```

###### logTagKey
* 定义：代表一个TagKey, 包含其对应的所的的tag value
```
type logTagKey struct {
	name      []byte
	deleted   bool
	tagValues map[string]logTagValue
}
```
* 添加TagValue
```
func (tk *logTagKey) createTagValueIfNotExists(value []byte) logTagValue {
    //这个value就是tag value的具体值，它作为key
	tv, ok := tk.tagValues[string(value)]
	if !ok {
		tv = logTagValue{name: value, series: make(map[uint64]struct{})}
	}
	return tv
}
```
* 生成TagValueIterator, 用来遍历这个TagKey对应的所有的TagValue
```
func (tk *logTagKey) TagValueIterator() TagValueIterator {
	a := make([]logTagValue, 0, len(tk.tagValues))
	for _, v := range tk.tagValues {
		a = append(a, v)
	}
	return newLogTagValueIterator(a)
}
```

###### logMeasurement
* 定义： 包含了一个measurement所有的tag key和series id
```
type logMeasurement struct {
	name      []byte  // measurement名字
	tagSet    map[string]logTagKey // tagkey的集合
	deleted   bool
	series    map[uint64]struct{} 
	seriesSet *tsdb.SeriesIDSet
}
```
其中`series`和`seriesSet`其实都是用来存储SeriesID, 当SeriesID数量小于等于25个时，存到`series`里，反之存到`seriesSet`这个roaring bitmap里;
* 添加新的tagkey
```
func (m *logMeasurement) createTagSetIfNotExists(key []byte) logTagKey {
	ts, ok := m.tagSet[string(key)]
	if !ok {
		ts = logTagKey{name: key, tagValues: make(map[string]logTagValue)}
	}
}
```
* 针对一个query请求，比如`select * from measurement1 where tag1=tv1 and tag2=tv2`, 根据`measurement1`我们可以确定到一个具体的`logMeasurement`, 然后根据`tag1=tv1`和`tag2=tv2`中的tagkey我们可以在`logMeasurement.tagSet`中锁定`logTagKey`, 在`logTagKey`中我们根据tagvalue就可以取到对应的一系列series id啦～注意，这里就是所谓的纯内存的倒排索引
* `logMeasureMent`的管理，包括创建，查询到操作是在`logFile`对象里完成，我们稍后会介绍.

###### LogEntry
* 定义： 写入到WAL文件中的数据格式,实际上是写入 dbname/rp/id/index/[id]/Lx-xxxxxxxx.tsl文件
```
type LogEntry struct {
	Flag     byte   // flag
	SeriesID uint64 // series id
	Name     []byte // measurement name
	Key      []byte // tag key
	Value    []byte // tag value
	Checksum uint32 // checksum of flag/name/tags.
	Size     int    // total size of record, in bytes.
	//以上部分是LogEntry的真正的内容

	cached   bool        // Hint to LogFile that series data is already parsed
	name     []byte      // series naem, this is a cached copy of the parsed measurement name
	tags     models.Tags // series tags, this is a cached copied of the parsed tags
	batchidx int         // position of entry in batch.
}
```
* 提供了序列化和反序列化方法： `appendLogEntry` `UnmarshalBinary` ，都比较简单，不累述;

###### LogFile
* 定义：提供了内存的倒排索引和索引WAL的写入
```
type LogFile struct {
	id         int            // file sequence identifier
	data       []byte         // mmap
	file       *os.File       // writer
	w          *bufio.Writer  // buffered writer
	bufferSize int            // The size of the buffer used by the buffered writer
	nosync     bool           // Disables buffer flushing and file syncing. Useful for offline tooling.
	buf        []byte         // marshaling buffer
	keyBuf     []byte

	sfile   *tsdb.SeriesFile // series lookup
	size    int64            // tracks current file size
	modTime time.Time        // tracks last time write occurred

	// In-memory series existence/tombstone sets.
	seriesIDSet, tombstoneSeriesIDSet *tsdb.SeriesIDSet

	// In-memory index.
	mms logMeasurements

	// Filepath to the log file.
	path string
}
```
* 添加新的`logMeasurement`, 添加到`LogFile.mms`, 它的类型是`type logMeasurements map[string]*logMeasurement`,key就是measurement name
```
func (f *LogFile) createMeasurementIfNotExists(name []byte) *logMeasurement {
	mm := f.mms[string(name)]
	if mm == nil {
		mm = &logMeasurement{
			name:   name,
			tagSet: make(map[string]logTagKey),
			series: make(map[uint64]struct{}),
		}
		f.mms[string(name)] = mm
	}
	return mm
}
```
* open操作
```
func (f *LogFile) open() error {
    //打开文件，为append操作准备
	file, err := os.OpenFile(f.Path(), os.O_WRONLY|os.O_CREATE, 0666)
	if err != nil {
		return err
	}
	f.file = file

    ...
	
	f.w = bufio.NewWriterSize(f.file, f.bufferSize)

	// 使用mmap映射现有文件内容到内存
	data, err := mmap.Map(f.Path(), 0)
	if err != nil {
		return err
	}
	f.data = data

	// 解析文件中的每一条logEntry,同时创建内存索引
	var n int64
	for buf := f.data; len(buf) > 0; {
		// Read next entry. Truncate partial writes.
		var e LogEntry
		// 反序列化成LogEntry对象
		if err := e.UnmarshalBinary(buf); err == io.ErrShortBuffer || err == ErrLogEntryChecksumMismatch {
			break
		} else if err != nil {
			return err
		}

		// 真正干事儿的都在这里
		f.execEntry(&e)

		// Move buffer forward.
		n += int64(e.Size)
		buf = buf[e.Size:]
	}

	//移动文件指针到末尾，准备写新数据
	f.size = n
	_, err = file.Seek(n, io.SeekStart)
	return err
}
```
* 启动时处理tsl文件中的每条LogEntry, 构建内存索引 或者运行时更新内存索引
```
func (f *LogFile) execEntry(e *LogEntry) {
	switch e.Flag {
	case LogEntryMeasurementTombstoneFlag:
		f.execDeleteMeasurementEntry(e)
	case LogEntryTagKeyTombstoneFlag:
		f.execDeleteTagKeyEntry(e)
	case LogEntryTagValueTombstoneFlag:
		f.execDeleteTagValueEntry(e)
	default:
		f.execSeriesEntry(e)
	}
}
```
这里`LogEntryMeasurementTombstoneFlag` `LogEntryTagKeyTombstoneFlag` `LogEntryTagValueTombstoneFlag`都是创建用于delete的`logMeasurement`对象,已经存在则更新相应的字段
* 处理单条Series操作 `execSeriesEntry`,看着代码多，其实很简单
```
func (f *LogFile) execSeriesEntry(e *LogEntry) {
	var seriesKey []byte
	if e.cached {
	    //将f.keyBuf更新为可以容纳最长的series key
		sz := tsdb.SeriesKeySize(e.name, e.tags)
		if len(f.keyBuf) < sz {
			f.keyBuf = make([]byte, 0, sz)
		}
		seriesKey = tsdb.AppendSeriesKey(f.keyBuf[:0], e.name, e.tags)
	} else {
	    // 从 series file里获取SeriesKey
		seriesKey = f.sfile.SeriesKey(e.SeriesID)
	}

	// Series keys can be removed if the series has been deleted from
	// the entire database and the server is restarted. This would cause
	// the log to replay its insert but the key cannot be found.
	//
	// https://github.com/influxdata/influxdb/issues/9444
	if seriesKey == nil {
		return
	}

    // 下面就都是解析这个 SeriesKey, 得到measurement, tag key , tag value
	// Check if deleted.
	deleted := e.Flag == LogEntrySeriesTombstoneFlag

	// Read key size.
	_, remainder := tsdb.ReadSeriesKeyLen(seriesKey)

	// Read measurement name.
	name, remainder := tsdb.ReadSeriesKeyMeasurement(remainder)
	mm := f.createMeasurementIfNotExists(name)
	mm.deleted = false
	if !deleted {
		mm.addSeriesID(e.SeriesID)
	} else {
		mm.removeSeriesID(e.SeriesID)
	}

	// Read tag count.
	tagN, remainder := tsdb.ReadSeriesKeyTagN(remainder)

	// Save tags.
	var k, v []byte
	for i := 0; i < tagN; i++ {
		k, v, remainder = tsdb.ReadSeriesKeyTag(remainder)
		ts := mm.createTagSetIfNotExists(k)
		tv := ts.createTagValueIfNotExists(v)

		// Add/remove a reference to the series on the tag value.
		if !deleted {
			tv.addSeriesID(e.SeriesID)
		} else {
			tv.removeSeriesID(e.SeriesID)
		}

		ts.tagValues[string(v)] = tv
		mm.tagSet[string(k)] = ts
	}

	// Add/remove from appropriate series id sets.
	if !deleted {
		f.seriesIDSet.Add(e.SeriesID)
		f.tombstoneSeriesIDSet.Remove(e.SeriesID)
	} else {
		f.seriesIDSet.Remove(e.SeriesID)
		f.tombstoneSeriesIDSet.Add(e.SeriesID)
	}
}
```
* 删除整个Measurement相关的索引, 先appEntry到tsl文件，成功后再更新内存索引
```
func (f *LogFile) DeleteMeasurement(name []byte) error {
	f.mu.Lock()
	defer f.mu.Unlock()

	e := LogEntry{Flag: LogEntryMeasurementTombstoneFlag, Name: name}
	if err := f.appendEntry(&e); err != nil {
		return err
	}
	f.execEntry(&e)

	// Flush buffer and sync to disk.
	return f.FlushAndSync()
}
```
类似的操作还有 `DeleteTagKey` `DeleteTagValue` `DeleteSeriesID`
* 获取到SeriesIDIterator, 用于遍历给定的tag key所对应的所有的tag value所在的每一个series id
```
func (f *LogFile) TagKeySeriesIDIterator(name, key []byte) tsdb.SeriesIDIterator {
	f.mu.RLock()
	defer f.mu.RUnlock()

	mm, ok := f.mms[string(name)]
	if !ok {
		return nil
	}

	tk, ok := mm.tagSet[string(key)]
	if !ok {
		return nil
	}

	// Combine iterators across all tag keys.
	itrs := make([]tsdb.SeriesIDIterator, 0, len(tk.tagValues))
	for _, tv := range tk.tagValues {
		if tv.cardinality() == 0 {
			continue
		}
		if itr := tsdb.NewSeriesIDSetIterator(tv.seriesIDSet()); itr != nil {
			itrs = append(itrs, itr)
		}
	}

	return tsdb.MergeSeriesIDIterators(itrs...)
}
```
* 批量添加SeriesKey，对于已经存在的就不处理，同时更新内存索引和写入tsl文件
```
func (f *LogFile) AddSeriesList(seriesSet *tsdb.SeriesIDSet, names [][]byte, tagsSlice []models.Tags) ([]uint64, error) {
    // 写入series file文件，返回所有的series id 列表
	seriesIDs, err := f.sfile.CreateSeriesListIfNotExists(names, tagsSlice)
	if err != nil {
		return nil, err
	}

	var writeRequired bool
	entries := make([]LogEntry, 0, len(names))
	seriesSet.RLock()
	for i := range names {
	    // seriesSet是该函数传进来的第一个参数，如果id已经存在于这个给定的seriesSet中，就不处理当前的id
		if seriesSet.ContainsNoLock(seriesIDs[i]) {
			// We don't need to allocate anything for this series.
			seriesIDs[i] = 0
			continue
		}
		writeRequired = true
		// 添充后面要使用的LogEntry列表
		entries = append(entries, LogEntry{SeriesID: seriesIDs[i], name: names[i], tags: tagsSlice[i], cached: true, batchidx: i})
	}
	seriesSet.RUnlock()

	// Exit if all series already exist.
	if !writeRequired {
		return seriesIDs, nil
	}

	f.mu.Lock()
	defer f.mu.Unlock()

	seriesSet.Lock()
	defer seriesSet.Unlock()

	for i := range entries { // NB - this doesn't evaluate all series ids returned from series file.
		entry := &entries[i]
	    //  上面已经过滤过一次了，这里还需要再过滤吗?
		if seriesSet.ContainsNoLock(entry.SeriesID) {
			// We don't need to allocate anything for this series.
			seriesIDs[entry.batchidx] = 0
			continue
		}
		if err := f.appendEntry(entry); err != nil {
			return nil, err
		}
		f.execEntry(entry)
		seriesSet.AddNoLock(entry.SeriesID)
	}

	// Flush buffer and sync to disk.
	if err := f.FlushAndSync(); err != nil {
		return nil, err
	}
	return seriesIDs, nil
}
```