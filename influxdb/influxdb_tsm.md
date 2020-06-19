[toc]

## Influxdb中TSM文件结构解析之WAL

#### 存储在Influxdb中的数据类型
###### 存储每条数据时的时间戳类型
* time 

###### Field字段的类型
* interger - int64
* unsigned - uint64
* float64
* boolean
* string

###### Field字段的类型在源码中对应类型
对应的类型是`Value`,这是个interface，定义在`tsdb/engine/tsm1/encoding.go`中
* IntegerValue
* UnsignedValue
* FloatValue
* BooleanValue
* StringValue
上面的每个类型都包括一个时间戳，这个时间戳就是这个数据被写入时的时间戳，我们看一下`FloadValue`的定义：
```
type FloatValue struct {
	unixnano int64
	value    float64
}
```

###### 编解码
每种类型在存储时都需要作编码，尽可能地作压缩，所有针对各个类型均提供了Encoder和Decoder。
这些Encoder负责将一组相同类型的`Value`作压缩编码，具体的编码算法我们这里不再展开。
我们针对`FloatValue`作一下分析`encodeFloatBlockUsing`
参数中`values []Value就是一系列的`FloatValue`,不仅包括Float值，还包括对应的时间戳，都需要被编码
```
func encodeFloatBlockUsing(buf []byte, values []Value, tsenc TimeEncoder, venc *FloatEncoder) ([]byte, error) {
	tsenc.Reset()
	venc.Reset()

	for _, v := range values {
		vv := v.(FloatValue)
		tsenc.Write(vv.unixnano) //使用TimeEncoder编码每个时间戳
		venc.Write(vv.value) //使用FloatEncoder编码每个Float值
	}
	venc.Flush()

	// Encoded timestamp values
	tb, err := tsenc.Bytes()
	if err != nil {
		return nil, err
	}
	// Encoded float values
	vb, err := venc.Bytes()
	if err != nil {
		return nil, err
	}

	// Prepend the first timestamp of the block in the first 8 bytes and the block
	// in the next byte, followed by the block
	// 将这一组FloatValue打包到一个Block
	return packBlock(buf, BlockFloat64, tb, vb), nil
}
```

###### 打包到DataBlock
DataBlock是写入和读取TSM文件的最小单位，每个DataBlock里存储的都是同样类型的Value,每个DataBlock里的Value对应都是同一个写入的Key,这个Key是`series key + field`;
Influxdb算是列存储，在这里所有的Value是连续存在一起，这些Value对应的时间戳也是连续存在一起，这样更有利于作压缩

<div align=center>![kafka-single-datacenter](file:///home/lw/docs/influxdb/influxdb_data_block.png)</div>

这个结构中并没有记录Values部分的长度，这是因为我们记录了时间戳部分的总长，在解析时间戳部分时候我们可以得知有几个时间戳，也就知道了有几个Value。

我们来看一下打包过程,结合上面的结构图，这个过程就很简单了：
```
func packBlock(buf []byte, typ byte, ts []byte, values []byte) []byte {
	// We encode the length of the timestamp block using a variable byte encoding.
	// This allows small byte slices to take up 1 byte while larger ones use 2 or more.
	sz := 1 + binary.MaxVarintLen64 + len(ts) + len(values)
	if cap(buf) < sz {
		buf = make([]byte, sz)
	}
	b := buf[:sz]
	b[0] = typ
	i := binary.PutUvarint(b[1:1+binary.MaxVarintLen64], uint64(len(ts)))
	i += 1

	// block is <len timestamp bytes>, <ts bytes>, <value bytes>
	copy(b[i:], ts)
	// We don't encode the value length because we know it's the rest of the block after
	// the timestamp block.
	copy(b[i+len(ts):], values)
	return b[:i+len(ts)+len(values)]
}
```

###### 解包DataBlock
我们还以`FloatValue`为例
```
func DecodeFloatBlock(block []byte, a *[]FloatValue) ([]FloatValue, error) {
	// Block type is the next block, make sure we actually have a float block
	blockType := block[0]
	if blockType != BlockFloat64 {
		return nil, fmt.Errorf("invalid block type: exp %d, got %d", BlockFloat64, blockType)
	}
	
	// 跳过1字节的block type
	block = block[1:]

	tb, vb, err := unpackBlock(block)
	if err != nil {
		return nil, err
	}

    //计算有多少组Value
	sz := CountTimestamps(tb)

	if cap(*a) < sz {
		*a = make([]FloatValue, sz)
	} else {
		*a = (*a)[:sz]
	}

	tdec := timeDecoderPool.Get(0).(*TimeDecoder)
	vdec := floatDecoderPool.Get(0).(*FloatDecoder)

	var i int
	err = func(a []FloatValue) error {
		// Setup our timestamp and value decoders
		tdec.Init(tb)
		err = vdec.SetBytes(vb)
		if err != nil {
			return err
		}

		// Decode both a timestamp and value
		j := 0
		for j < len(a) && tdec.Next() && vdec.Next() {
			a[j] = FloatValue{unixnano: tdec.Read(), value: vdec.Values()}
			j++
		}
		i = j

		// Did timestamp decoding have an error?
		err = tdec.Error()
		if err != nil {
			return err
		}

		// Did float decoding have an error?
		return vdec.Error()
	}(*a)
	
		timeDecoderPool.Put(tdec)
	floatDecoderPool.Put(vdec)

	return (*a)[:i], err
```

###### Dabablock的其他操作
* `BlockType`:取block byte buffer的第一个字节
```
func BlockType(block []byte) (byte, error) {
	blockType := block[0]
	switch blockType {
	case BlockFloat64, BlockInteger, BlockUnsigned, BlockBoolean, BlockString:
		return blockType, nil
	default:
		return 0, fmt.Errorf("unknown block type: %d", blockType)
	}
}
```

* `BlockCount`: 获取一个DabaBlock中包含的Value数量
```
func BlockCount(block []byte) int {
	if len(block) <= encodedBlockHeaderSize {
		panic(fmt.Sprintf("count of short block: got %v, exp %v", len(block), encodedBlockHeaderSize))
	}
	// first byte is the block type
	tb, _, err := unpackBlock(block[1:])
	if err != nil {
		panic(fmt.Sprintf("BlockCount: error unpacking block: %s", err.Error()))
	}
	return CountTimestamps(tb)
}
```

* `DecodeBlock`: 解码一个DabaBlock,根据BlockType的不同调用不同的Decode方法
```
func DecodeBlock(block []byte, vals []Value) ([]Value, error) {
	if len(block) <= encodedBlockHeaderSize {
		panic(fmt.Sprintf("decode of short block: got %v, exp %v", len(block), encodedBlockHeaderSize))
	}

	blockType, err := BlockType(block)
	if err != nil {
		return nil, err
	}

	switch blockType {
	case BlockFloat64:
		var buf []FloatValue
		decoded, err := DecodeFloatBlock(block, &buf)
		if len(vals) < len(decoded) {
			vals = make([]Value, len(decoded))
		}
		for i := range decoded {
			vals[i] = decoded[i]
		}
		return vals[:len(decoded)], err
	case BlockInteger:
		...
	case BlockUnsigned:
		...
	case BlockBoolean:
		...
	case BlockString:
		...
	default:
		panic(fmt.Sprintf("unknown block type: %d", blockType))
	}
}
```

#### WALEntry
1. WAL在写入TSM文件时用作预写日志。
2. 每个DB的每个RetentionPolicy下面的每个Shard下都有自己的一个单独的WAL文件目录，Influxdb在启动的配置文件中需设置单独的WAL目录，来存储所有Shard的WAL文件。
3. 每个Shard都对应一个WAL目录，目录下有多个wal文件，每个称作一个`WALSegment`,默认大小是10M。文件命名规则是，以`_`开头，中间是ID，扩展名是`wal`, 比如 `_00001.wal`
4. 每次写入WAL的内容称为一个`WALEntry`, 在写入和读取这个Entry时需要序列化和反序列化,我们先来看一下其定义：
```
type WALEntry interface {
	Type() WalEntryType  // Entry的类型： WriteWALEntry， DeleteWALEntry, DeleteRangeWALEntry
	Encode(dst []byte) ([]byte, error)
	MarshalBinary() ([]byte, error) //使用上面的Encode方法作序列化
	UnmarshalBinary(b []byte) error //反序列化
	MarshalSize() int
}
```
我们下面来分析一下具体的三种`WALEntry`

###### WriteWALEntry
* 一组`point`组成一个`WriteEALEntry`,然后写入`WALSegment`;
`point`是一个series对应的一些field的集合，每个`point`被唯一的`series` + `timestamp`标识，可以简单将`point`理解为就是一个`insert`语句写入的内容。
* 定义：
```
type WriteWALEntry struct {
	Values map[string][]Value
	sz     int
}
```
其中`Valuse`是个map,它的`key`是series key + field, 它的`value`是具有相同的key的所有field value;其实就是把多个point按series key + field作了合并
* 结构图

<div align=center>![kafka-single-datacenter](file:///home/lw/docs/influxdb/influxdb_write_wal_entry.png)</div>

* 序列化`Encode`：完全按照上面的结构图来写入,比较清晰明了
```
func (w *WriteWALEntry) Encode(dst []byte) ([]byte, error) {
    // 计算总大小，欲分配内存
	encLen := w.MarshalSize() // Type (1), Key Length (2), and Count (4) for each key

	// allocate or re-slice to correct size
	if len(dst) < encLen {
		dst = make([]byte, encLen)
	} else {
		dst = dst[:encLen]
	}

	// Finally, encode the entry
	var n int
	var curType byte

    // 遍历Values,逐一编码
	for k, v := range w.Values {
	    // 确定field的类型
		switch v[0].(type) {
		case FloatValue:
			curType = float64EntryType
		case IntegerValue:
			curType = integerEntryType
		case UnsignedValue:
			curType = unsignedEntryType
		case BooleanValue:
			curType = booleanEntryType
		case StringValue:
			curType = stringEntryType
		default:
			return nil, fmt.Errorf("unsupported value type: %T", v[0])
		}
		
		// 写入类型
		dst[n] = curType
		n++
 
        // 写入key长度，key = series key + field
		binary.BigEndian.PutUint16(dst[n:n+2], uint16(len(k)))
		n += 2
		// 写入 key
		n += copy(dst[n:], k)

        // 写入 value个数
		binary.BigEndian.PutUint32(dst[n:n+4], uint32(len(v)))
		n += 4

        // 逐一写入合部的value
		for _, vv := range v {
			binary.BigEndian.PutUint64(dst[n:n+8], uint64(vv.UnixNano()))
			n += 8

			switch vv := vv.(type) {
			case FloatValue:
				if curType != float64EntryType {
					return nil, fmt.Errorf("incorrect value found in %T slice: %T", v[0].Value(), vv)
				}
				binary.BigEndian.PutUint64(dst[n:n+8], math.Float64bits(vv.value))
				n += 8
			case IntegerValue:
				if curType != integerEntryType {
					return nil, fmt.Errorf("incorrect value found in %T slice: %T", v[0].Value(), vv)
				}
				binary.BigEndian.PutUint64(dst[n:n+8], uint64(vv.value))
				n += 8
			case UnsignedValue:
				if curType != unsignedEntryType {
					return nil, fmt.Errorf("incorrect value found in %T slice: %T", v[0].Value(), vv)
				}
				binary.BigEndian.PutUint64(dst[n:n+8], uint64(vv.value))
				n += 8
			case BooleanValue:
				if curType != booleanEntryType {
					return nil, fmt.Errorf("incorrect value found in %T slice: %T", v[0].Value(), vv)
				}
				if vv.value {
					dst[n] = 1
				} else {
					dst[n] = 0
				}
				n++
			case StringValue:
				if curType != stringEntryType {
					return nil, fmt.Errorf("incorrect value found in %T slice: %T", v[0].Value(), vv)
				}
				binary.BigEndian.PutUint32(dst[n:n+4], uint32(len(vv.value)))
				n += 4
				n += copy(dst[n:], vv.value)
			default:
				return nil, fmt.Errorf("unsupported value found in %T slice: %T", v[0].Value(), vv)
			}
		}
	}

	return dst[:n], nil
}
```

###### DeleteWALEntry
* 删除Series的WALEntry
* 定义：
```
type DeleteWALEntry struct {
	Keys [][]byte
	sz   int
}
```
* 结构图

<div align=center>![kafka-single-datacenter](file:///home/lw/docs/influxdb/influxdb_wal_delete_entry.png)</div>

* 编码`Encode`, 各个key间以`\n`分隔
```
func (w *DeleteWALEntry) Encode(dst []byte) ([]byte, error) {
	sz := w.MarshalSize()

	if len(dst) < sz {
		dst = make([]byte, sz)
	}

	var n int
	for _, k := range w.Keys {
		n += copy(dst[n:], k)
		n += copy(dst[n:], "\n")
	}

	// We return n-1 to strip off the last newline so that unmarshalling the value
	// does not produce an empty string
	return []byte(dst[:n-1]), nil
}
```

###### DeleteRangeWALEntry
* 删除某个时间范围内的series的WALEntry
* 定义：
```
type DeleteRangeWALEntry struct {
	Keys     [][]byte
	Min, Max int64  // 开始时间戳和结束时间戳
	sz       int
}
```

* 结构图

<div align=center>![kafka-single-datacenter](file:///home/lw/docs/influxdb/influxdb_delete_ragne_wal_entry.png)</div>

* 编码`Encode`
```
func (w *DeleteRangeWALEntry) Encode(b []byte) ([]byte, error) {
	sz := w.MarshalSize()

	if len(b) < sz {
		b = make([]byte, sz)
	}

    // 写入开始和结束时间戳
	binary.BigEndian.PutUint64(b[:8], uint64(w.Min))
	binary.BigEndian.PutUint64(b[8:16], uint64(w.Max))

	i := 16
	// 逐一写入key
	for _, k := range w.Keys {
		binary.BigEndian.PutUint32(b[i:i+4], uint32(len(k)))
		i += 4
		i += copy(b[i:], k)
	}

	return b[:i], nil
}
```

#### WALEntry的写入
* 上面我们介绍了三种WALEntry,在序列化后就可以被写入到WALSegment文件中了，在写之前可能还需要作压缩
* 写入时候为了读取时便于解析，还需要按一定格式写入 
  1. 先写入 1字节 的 WALEntry类型
  2. 再写入 4字节 的 序列化后且作了压缩的WALEntry的长度
  3. 最后写入 序列化后且作了压缩的WALEntry的具体内容
* 使用 `WALSegmentWriter`类来写入:
```
func (w *WALSegmentWriter) Write(entryType WalEntryType, compressed []byte) error {
	var buf [5]byte
	// 写入类型和具体内容的长度
	buf[0] = byte(entryType)
	binary.BigEndian.PutUint32(buf[1:5], uint32(len(compressed)))

	if _, err := w.bw.Write(buf[:]); err != nil {
		return err
	}

    // 写入具体内容
	if _, err := w.bw.Write(compressed); err != nil {
		return err
	}

	w.size += len(buf) + len(compressed)

	return nil
}
```

#### WAL
WAL封装了一个预写日志的所有操作，正如前面提到了，一个Shard对应一个WAL,一个WAL在写入时又会产生多个WALSegment。
我们来分析一下一些主要的方法:
###### `Open`操作
遍历一个Shard目录下的所有Segment文件，这些文件按id从小到大排序，作初始化操作
```
func (l *WAL) Open() error {
	l.mu.Lock()
	defer l.mu.Unlock()
..

	if err := os.MkdirAll(l.path, 0777); err != nil {
		return err
	}

    // 获取所有segment 文件列表，按id从小到大排序，最后一个就是当前正写入的文件 
	segments, err := segmentFileNames(l.path)
	if err != nil {
		return err
	}

	if len(segments) > 0 {
	    // 最后一个就是当前正写入的文件
		lastSegment := segments[len(segments)-1]
		
		// 获取最新的segment id
		id, err := idFromFileName(lastSegment)
		if err != nil {
			return err
		}

        // 初始化当前的segment id
		l.currentSegmentID = id
		stat, err := os.Stat(lastSegment)
		if err != nil {
			return err
		}

		if stat.Size() == 0 {
		    // 如果文件大小为0, 删除
			os.Remove(lastSegment)
			segments = segments[:len(segments)-1]
		} else {
		    //为写入，打开该文件 
			fd, err := os.OpenFile(lastSegment, os.O_RDWR, 0666)
			if err != nil {
				return err
			}
			if _, err := fd.Seek(0, io.SeekEnd); err != nil {
				return err
			}
			
			// 初始化当前的SegmentWriter
			l.currentSegmentWriter = NewWALSegmentWriter(fd)

			// Reset the current segment size stat
			atomic.StoreInt64(&l.stats.CurrentBytes, stat.Size())
		}
	}

	...
	
	l.closing = make(chan struct{})

	return nil
}
```

######  `writeToLog`写入操作
```
func (l *WAL) writeToLog(entry WALEntry) (int, error) {
    // 从buytesPool获取byte slice, 避免反复重新分配内存
	bytes := bytesPool.Get(entry.MarshalSize())

    // 将entry作编码，前面已经介绍过
	b, err := entry.Encode(bytes)
	if err != nil {
		bytesPool.Put(bytes)
		return -1, err
	}

    // 使用snappy压缩强词编码后的entry内容
	encBuf := bytesPool.Get(snappy.MaxEncodedLen(len(b)))

	compressed := snappy.Encode(encBuf, b)
	bytesPool.Put(bytes)

	syncErr := make(chan error)

	segID, err := func() (int, error) {
		l.mu.Lock()
		defer l.mu.Unlock()

		// Make sure the log has not been closed
		select {
		case <-l.closing:
			return -1, ErrWALClosed
		default:
		}

		// roll the segment file if needed
		if err := l.rollSegment(); err != nil {
			return -1, fmt.Errorf("error rolling WAL segment: %v", err)
		}

		// write and sync
		// 使用SegmentWriter来写入entry内容
		if err := l.currentSegmentWriter.Write(entry.Type(), compressed); err != nil {
			return -1, fmt.Errorf("error writing WAL entry: %v", err)
		}

		select {
		case l.syncWaiters <- syncErr:
		default:
			return -1, fmt.Errorf("error syncing wal")
		}
		
		// 将执行file sync操作，刷到磁盘文件 
		l.scheduleSync()

		// Update stats for current segment size
		atomic.StoreInt64(&l.stats.CurrentBytes, int64(l.currentSegmentWriter.size))

		l.lastWriteTime = time.Now().UTC()

		return l.currentSegmentID, nil
	}()

	bytesPool.Put(encBuf)

	if err != nil {
		return segID, err
	}

	// schedule an fsync and wait for it to complete
	return segID, <-syncErr
}
```

#### Cache
1. 时序数据在写入时，会先写入到上面介绍的WAL,然后写入到Cache,最后按照一定的策略Flush到磁盘文件。现在我们来介绍这个Cache。
2. 这个Cache里缓存的是什么呢？
3. 这个Cache用什么结构来作内存存储？
我们下面来一一解答这些问题：

###### Entry
* 既然是Cache，那肯定是key-value结构，其中的key是`series key + field name`, 对应的value就是这个key所对应的若干个field value的集合，也就组合成了一个entry,我们来看下entry的定义:
```
type entry struct {
	mu     sync.RWMutex
	values Values // All stored values.

	// The type of values stored. Read only so doesn't need to be protected by
	// mu.
	vtype byte
}
```
由这个定义我们可知，同一个entry里面的所有`value`的类型都是相同的，都是这个 `vtype`里所保存的类型。

* `Entry`的创建：使用`[]Value`来创建，比较简单，但需要先判断这组value的类型是否一致
```
func newEntryValues(values []Value) (*entry, error) {
	e := &entry{}
	e.values = make(Values, 0, len(values))
	e.values = append(e.values, values...)

	// No values, don't check types and ordering
	if len(values) == 0 {
		return e, nil
	}

    // 个人感觉应该先校验这组value的类型是否一致，不一致就不要作上面的make, append了。
	et := valueType(values[0])
	for _, v := range values {
		// Make sure all the values are the same type
		if et != valueType(v) {
			return nil, tsdb.ErrFieldTypeConflict
		}
	}

	// Set the type of values stored.
	e.vtype = et

	return e, nil
}
```

* `Entry`的add操作：和上面的创建类似，需要先判断这组value的类型是否一致
*  `Entry`的去重操作，去掉`Values`中时间戳相同的value,只保留其中的一个
```
func (e *entry) deduplicate() {
	e.mu.Lock()
	defer e.mu.Unlock()

	if len(e.values) <= 1 {
		return
	}
	e.values = e.values.Deduplicate()
}
```
实际上是调用了`Values.Deduplicate`，这个`Values`提供了若干实用的方法，比如去掉，过滤等。

* `Entry`过滤：过滤掉在给定时间戳范围内的Value
```
func (e *entry) filter(min, max int64) {
	e.mu.Lock()
	if len(e.values) > 1 {
		e.values = e.values.Deduplicate()
	}
	e.values = e.values.Exclude(min, max)
	e.mu.Unlock()
}
```
实际上是调用了`Values.Exclude`

###### storer
* 上面解决了Cache存什么的问题，下面我们来解决怎么存的问题。这个存储器是`storer`,它是个interface,之前是实现了这个interface的struct都可以用来存Cache里的entry,我们先来看一下这个interface
```
type storer interface {
	entry(key []byte) *entry                        // Get an entry by its key.
	write(key []byte, values Values) (bool, error)  // Write an entry to the store.
	add(key []byte, entry *entry)                   // Add a new entry to the store.
	remove(key []byte)                              // Remove an entry from the store.
	keys(sorted bool) [][]byte                      // Return an optionally sorted slice of entry keys.
	apply(f func([]byte, *entry) error) error       // Apply f to all entries in the store in parallel.
	applySerial(f func([]byte, *entry) error) error // Apply f to all entries in serial.
	reset()                                         // Reset the store to an initial unused state.
	split(n int) []storer                           // Split splits the store into n stores
	count() int                                     // Count returns the number of keys in the store
}
```
注释很清晰，我们这里不累述。
* 那实际是用什么来存的呢？ influxdb里实现了`ring`,它实现了这个`storer`的所有接口,定义在`tsdb/engine/tsm1/ring.go`中。简单来说，一个ring内部分为若干个固定数量的桶，这里叫`partition`, 一个key进来后，按key作hash，对桶的数量取模，确定好要存在哪个桶里，然后每个桶其实又是一个map,最后也就是将key-value存在这个桶的map里。因为cache的添加，读取可能很频繁且都需要加锁，分桶后，各个桶单独加锁，提升性能。代码很简单，这里不详述了。