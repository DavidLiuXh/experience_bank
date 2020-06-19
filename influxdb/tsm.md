
#### TSM文件组成概述
* 每个TSM文件由4部分组成，源码里给出了文件结构，我们在这里搬过来
Header, Blocks, Index, Footer
```
┌────────┬────────────────────────────────────┬─────────────┬──────────────┐
│ Header │               Blocks               │    Index    │    Footer    │
│5 bytes │              N bytes               │   N bytes   │   4 bytes    │
└────────┴────────────────────────────────────┴─────────────┴──────────────┘
```
* Header部分：magic number + version
```
┌───────────────────┐
│      Header       │
├─────────┬─────────┤
│  Magic  │ Version │
│ 4 bytes │ 1 byte  │
└─────────┴─────────┘
```
* Blocks部分： 若干个block的集合，这个就是我们前面介绍过的DataBlock, 每个block都有对应的crc校验
```
┌───────────────────────────────────────────────────────────┐
│                          Blocks                           │
├───────────────────┬───────────────────┬───────────────────┤
│      Block 1      │      Block 2      │      Block N      │
├─────────┬─────────┼─────────┬─────────┼─────────┬─────────┤
│  CRC    │  Data   │  CRC    │  Data   │  CRC    │  Data   │
│ 4 bytes │ N bytes │ 4 bytes │ N bytes │ 4 bytes │ N bytes │
└─────────┴─────────┴─────────┴─────────┴─────────┴─────────┘
```
* Index部分： 按key从小到大排续，每个key的Index又包含多个IndexEntry,每个IndexEntry通过offset和size指向其DataBlock的位置
```
┌────────────────────────────────────────────────────────────────────────────┐
│                                   Index                                    │
├─────────┬─────────┬──────┬───────┬─────────┬─────────┬────────┬────────┬───┤
│ Key Len │   Key   │ Type │ Count │Min Time │Max Time │ Offset │  Size  │...│
│ 2 bytes │ N bytes │1 byte│2 bytes│ 8 bytes │ 8 bytes │8 bytes │4 bytes │   │
└─────────┴─────────┴──────┴───────┴─────────┴─────────┴────────┴────────┴───┘
│ Key Len │   Key   │ Type │ Count │Min Time │Max Time │ Offset │  Size  │...│
│ 2 bytes │ N bytes │1 byte│2 bytes│ 8 bytes │ 8 bytes │8 bytes │4 bytes │   │
└─────────┴─────────┴──────┴───────┴─────────┴─────────┴────────┴────────┴───┘
```
* Footer部分：保存有index的offset
```
┌─────────┐
│ Footer  │
├─────────┤
│Index Ofs│
│ 8 bytes │
└─────────┘
```
* 对这个TSM文件的读写都是依照上面的结构，我们下面分别来分析一下

#### TSM文件写操作
##### Index的数据结构
* Index部分的组成上面已经说过，可以简单认为Index部分由若干子index构成，key相同的IndexEntry构成一条子Index,这个Key就是`series key + field`, 每条IndexEntry都指向一个DataBlock。
* IndexEntry定义：
```
type IndexEntry struct {
	// The min and max time of all points stored in the block.
	MinTime, MaxTime int64

	// The absolute position in the file where this block is located.
	Offset int64

	// The size in bytes of the block in the file.
	Size uint32
}
```
这些IndexEntry也是按MinTime来排序存储的。
* 这样在查找数据因为key是排序的，可以先快速定位到相应的子index，然后IndexEntry中的`MinTime`和`MaxTime`来定位到具体的DabaBlock,每个DabaBlock中的`Values`也是带时间戳存储的，这样就查找到想要的数据了。
* 序列化操作`AppendTo`:
```
func (e *IndexEntry) AppendTo(b []byte) []byte {
	if len(b) < indexEntrySize {
		if cap(b) < indexEntrySize {
			b = make([]byte, indexEntrySize)
		} else {
			b = b[:indexEntrySize]
		}
	}

	binary.BigEndian.PutUint64(b[:8], uint64(e.MinTime))
	binary.BigEndian.PutUint64(b[8:16], uint64(e.MaxTime))
	binary.BigEndian.PutUint64(b[16:24], uint64(e.Offset))
	binary.BigEndian.PutUint32(b[24:28], uint32(e.Size))

	return b
}
```
* 反序列化操作`UnmarshalBinary`:
```
func (e *IndexEntry) UnmarshalBinary(b []byte) error {
	if len(b) < indexEntrySize {
		return fmt.Errorf("unmarshalBinary: short buf: %v < %v", len(b), indexEntrySize)
	}
	e.MinTime = int64(binary.BigEndian.Uint64(b[:8]))
	e.MaxTime = int64(binary.BigEndian.Uint64(b[8:16]))
	e.Offset = int64(binary.BigEndian.Uint64(b[16:24]))
	e.Size = binary.BigEndian.Uint32(b[24:28])
	return nil
}
```
##### 写入Index
* Index在写入文件前需要先缓存，这通过`directIndex`类来完成的。缓存有两种选择：内存和磁盘文件
```
type directIndex struct {
	keyCount int
	size     uint32

	// The bytes written count of when we last fsync'd
	lastSync uint32
	fd       *os.File       // 缓存到磁盘文件
	buf      *bytes.Buffer  // 缓存到内存Buffer

	f syncer

	w *bufio.Writer

    // 下面两项合起来代表一个子index
	key          []byte
	indexEntries *indexEntries  
}
```
写Index时是按照子index一个一个写入。
* `directIndex`的创建
 1. 创建基于内存的缓存:
 ```
 func NewIndexWriter() IndexWriter {
	buf := bytes.NewBuffer(make([]byte, 0, 1024*1024))
	return &directIndex{buf: buf, w: bufio.NewWriter(buf)}
}
 ```
 2. 创建基于Disk的缓存:
 ```
 func NewDiskIndexWriter(f *os.File) IndexWriter {
	return &directIndex{fd: f, w: bufio.NewWriterSize(f, 1024*1024)}
}
 ```
* `Add`添加新的Index:
`directIndex`中的`key`来缓存当前写入的子index的key值，`directIndex`中的`indexEntires`缓存当前key对应的所有IndexEntry
```
func (d *directIndex) Add(key []byte, blockType byte, minTime, maxTime int64, offset int64, size uint32) {
	// d.key为空时，写入这个新的key的length, key value等信息，相当于是一个新的子index的开始
	if len(d.key) == 0 {
		// size of the key stored in the index
		d.size += uint32(2 + len(key))
		// size of the count of entries stored in the index
		d.size += indexCountSize

		d.key = key
		if d.indexEntries == nil {
			d.indexEntries = &indexEntries{}
		}
		d.indexEntries.Type = blockType
		//添加新的IndexEntry到entries中
		d.indexEntries.entries = append(d.indexEntries.entries, IndexEntry{
			MinTime: minTime,
			MaxTime: maxTime,
			Offset:  offset,
			Size:    size,
		})

		// 记录index总的大小和不同key的个数的
		d.size += indexEntrySize
		d.keyCount++
		return
	}

	// See if were still adding to the same series key.
	cmp := bytes.Compare(d.key, key)
	if cmp == 0 {
		// key相同说明当前子index的添加还没有结束
		d.indexEntries.entries = append(d.indexEntries.entries, IndexEntry{
			MinTime: minTime,
			MaxTime: maxTime,
			Offset:  offset,
			Size:    size,
		})

		// size of the encoded index entry
		d.size += indexEntrySize

	} else if cmp < 0 {
	    //子index的写入是按key从小到大排序的，如果key不同了，说明上一个key对应的子index添加完成了，需要flush后开启一个新的子index的写入，这个flush动作很重要，我们下面还会细讲
		d.flush(d.w)
		// We have a new key that is greater than the last one so we need to add
		// a new index block section.

		// size of the key stored in the index
		d.size += uint32(2 + len(key))
		// size of the count of entries stored in the index
		d.size += indexCountSize

		d.key = key
		d.indexEntries.Type = blockType
		d.indexEntries.entries = append(d.indexEntries.entries, IndexEntry{
			MinTime: minTime,
			MaxTime: maxTime,
			Offset:  offset,
			Size:    size,
		})

		// size of the encoded index entry
		d.size += indexEntrySize
		d.keyCount++
	} else {
		// Keys can't be added out of order.
		panic(fmt.Sprintf("keys must be added in sorted order: %s < %s", string(key), string(d.key)))
	}
}
```
* flush操作：
```
func (d *directIndex) flush(w io.Writer) (int64, error) {
	var (
		n   int
		err error
		buf [5]byte
		N   int64
	)

	if len(d.key) == 0 {
		return 0, nil
	}
	// For each key, individual entries are sorted by time
	key := d.key
	entries := d.indexEntries

	if entries.Len() > maxIndexEntries {
		return N, fmt.Errorf("key '%s' exceeds max index entries: %d > %d", key, entries.Len(), maxIndexEntries)
	}

    // 将IndexEntry按MinTime来排序
	if !sort.IsSorted(entries) {
		sort.Sort(entries)
	}

	binary.BigEndian.PutUint16(buf[0:2], uint16(len(key)))
	buf[2] = entries.Type
	binary.BigEndian.PutUint16(buf[3:5], uint16(entries.Len()))

	// 写入key length
	if n, err = w.Write(buf[0:2]); err != nil {
		return int64(n) + N, fmt.Errorf("write: writer key length error: %v", err)
	}
	N += int64(n)

    // 写入key
	if n, err = w.Write(key); err != nil {
		return int64(n) + N, fmt.Errorf("write: writer key error: %v", err)
	}
	N += int64(n)

	// 写入类型和IndexEntry个数
	if n, err = w.Write(buf[2:5]); err != nil {
		return int64(n) + N, fmt.Errorf("write: writer block type and count error: %v", err)
	}
	N += int64(n)

	// 写每一个IndexEntry
	var n64 int64
	if n64, err = entries.WriteTo(w); err != nil {
		return n64 + N, fmt.Errorf("write: writer entries error: %v", err)
	}
	N += n64

	d.key = nil
	d.indexEntries.Type = 0
	d.indexEntries.entries = d.indexEntries.entries[:0]

	// 如果是基于磁盘文件的缓存，达到阈值后，Sync到磁盘文件 
	if d.fd != nil && d.size-d.lastSync > fsyncEvery {
		if err := d.fd.Sync(); err != nil {
			return N, err
		}
		d.lastSync = d.size
	}

	return N, nil
}
```

* `WriteTo`操作:
```
func (d *directIndex) WriteTo(w io.Writer) (int64, error) {
    // 先flush到d.w中
	if _, err := d.flush(d.w); err != nil {
		return 0, err
	}

    // 再从d.w中flush到内存buffer或文件
	if err := d.w.Flush(); err != nil {
		return 0, err
	}

	if d.fd == nil {
	    // 如果是缓存到内存，两个buffer作对拷
		return copyBuffer(d.f, w, d.buf, nil)
	}

    // 如果是缓存到磁盘,读磁盘文件拷贝到buffer,这个地方是否需要调用d.fd.Sync()?
	if _, err := d.fd.Seek(0, io.SeekStart); err != nil {
		return 0, err
	}

	return io.Copy(w, bufio.NewReaderSize(d.fd, 1024*1024))
}
```

##### TSM写入
* 通过`tsmWriter`来写入
```
type tsmWriter struct {
	wrapped io.Writer
	w       *bufio.Writer
	index   IndexWriter
	n       int64

	// The bytes written count of when we last fsync'd
	lastSync int64
}
```

* 写新的Dabablick
```
func (t *tsmWriter) Write(key []byte, values Values) error {
	if len(key) > maxKeyLength {
		return ErrMaxKeyLengthExceeded
	}

	// Nothing to write
	if len(values) == 0 {
		return nil
	}

	// Write header only after we have some data to write.
	if t.n == 0 {
	    // 先写头
		if err := t.writeHeader(); err != nil {
			return err
		}
	}

    // 将vluaes编码成Block
	block, err := values.Encode(nil)
	if err != nil {
		return err
	}

	blockType, err := BlockType(block)
	if err != nil {
		return err
	}

    // 计算crc
	var checksum [crc32.Size]byte
	binary.BigEndian.PutUint32(checksum[:], crc32.ChecksumIEEE(block))

    // 写入crc
	_, err = t.w.Write(checksum[:])
	if err != nil {
		return err
	}

    // 写入block
	n, err := t.w.Write(block)
	if err != nil {
		return err
	}
	n += len(checksum)

	// 写IndexEntry到Index缓存
	t.index.Add(key, blockType, values[0].UnixNano(), values[len(values)-1].UnixNano(), t.n, uint32(n))

	// Increment file position pointer
	t.n += int64(n)

	if len(t.index.Entries(key)) >= maxIndexEntries {
		return ErrMaxBlocksExceeded
	}

	return nil
}
```
* 写Index
```
func (t *tsmWriter) WriteIndex() error {
	indexPos := t.n

	if t.index.KeyCount() == 0 {
		return ErrNoValues
	}

	// Set the destination file on the index so we can periodically
	// fsync while writing the index.
	if f, ok := t.wrapped.(syncer); ok {
		t.index.(*directIndex).f = f
	}

	// Write the index
	if _, err := t.index.WriteTo(t.w); err != nil {
		return err
	}

	var buf [8]byte
	binary.BigEndian.PutUint64(buf[:], uint64(indexPos))

	// 写footer, 记录index在文件中的offset
	_, err := t.w.Write(buf[:])
	return err
}
```

#### TSM文件读取
###### mmapAccessor
* 采用mmap内存映射的方式读取tsm文件，先来看一下定义：
```
type mmapAccessor struct {
    // 下面这两个相当于引用计数
	accessCount uint64 // Counter incremented everytime the mmapAccessor is accessed
	freeCount   uint64 // Counter to determine whether the accessor can free its resources

    // mmap只是作了虚拟内存到磁盘文件地址的映射，建立了相应的页面，只有真正访问时，才从磁盘读入内存，如果这个标识为true,则在
	// 建立了映射后，使用advise系统调用建议os立即读到内存
	mmapWillNeed bool // If true then mmap advise value MADV_WILLNEED will be provided the kernel for b.

	mu sync.RWMutex
	b  []byte
	f  *os.File

    // tsm中的索引部分使用这个indirectIndex访问
	index *indirectIndex
}
```
* 初始化操作`init`:
```
func (m *mmapAccessor) init() (*indirectIndex, error) {
	m.mu.Lock()
	defer m.mu.Unlock()

	...

    // 内存映射当前的tsm文件
	m.b, err = mmap(m.f, 0, int(stat.Size()))
	if err != nil {
		return nil, err
	}

	// footer
	if len(m.b) < 8 {
		return nil, fmt.Errorf("mmapAccessor: byte slice too small for indirectIndex")
	}

	// Hint to the kernel that we will be reading the file.  It would be better to hint
	// that we will be reading the index section, but that's not been
	// implemented as yet.
	// 这个在上面的定义中已经作了解释
	if m.mmapWillNeed {
		if err := madviseWillNeed(m.b); err != nil {
			return nil, err
		}
	}

    // 读取tsm文件中的footer部分，获取index部分的偏移量
	indexOfsPos := len(m.b) - 8
	indexStart := binary.BigEndian.Uint64(m.b[indexOfsPos : indexOfsPos+8])
	if indexStart >= uint64(indexOfsPos) {
		return nil, fmt.Errorf("mmapAccessor: invalid indexStart")
	}

    // 创建indirectIndex对象（这个对象我们后面会专门介绍），读取并解析tsm中的index部分，获取到最小key,最大key,这里的key = series key + field
	// 当前tsm中最小时间戳和最大时间戳
	m.index = NewIndirectIndex()
	if err := m.index.UnmarshalBinary(m.b[indexStart:indexOfsPos]); err != nil {
		return nil, err
	}

	// Allow resources to be freed immediately if requested
	m.incAccess()
	atomic.StoreUint64(&m.freeCount, 1)

	return m.index, nil
}
```
* 读取Datablock `readBlock`:
```
func (m *mmapAccessor) readBlock(entry *IndexEntry, values []Value) ([]Value, error) {
	m.incAccess()

	m.mu.RLock()
	defer m.mu.RUnlock()

	if int64(len(m.b)) < entry.Offset+int64(entry.Size) {
		return nil, ErrTSMClosed
	}
	//TODO: Validate checksum
	var err error
	// 解析Datablock
	values, err = DecodeBlock(m.b[entry.Offset+4:entry.Offset+int64(entry.Size)], values)
	if err != nil {
		return nil, err
	}

	return values, nil
}
```
* 根据indexEntry读取crc和datablock,但不作datablock的解析
```
func (m *mmapAccessor) readBytes(entry *IndexEntry, b []byte) (uint32, []byte, error) {
	m.incAccess()

	m.mu.RLock()
	if int64(len(m.b)) < entry.Offset+int64(entry.Size) {
		m.mu.RUnlock()
		return 0, nil, ErrTSMClosed
	}

	// return the bytes after the 4 byte checksum
	crc, block := binary.BigEndian.Uint32(m.b[entry.Offset:entry.Offset+4]), m.b[entry.Offset+4:entry.Offset+int64(entry.Size)]
	m.mu.RUnlock()

	return crc, block, nil
}
```
* 读取所有的DataBlock
```
func (m *mmapAccessor) readAll(key []byte) ([]Value, error) {
	m.incAccess()

    //读出所有的datablock
	blocks := m.index.Entries(key)
	if len(blocks) == 0 {
		return nil, nil
	}

    // 获取当前key对应的已被标记为删除的时间段
	tombstones := m.index.TombstoneRange(key)

	m.mu.RLock()
	defer m.mu.RUnlock()

	var temp []Value
	var err error
	var values []Value
	for _, block := range blocks {
		var skip bool
		// 如果当前block的[minTimem, maxTime]是tombstones中的某个TimeRange的子集，表时当前这个block已可以全部被删除，不需要再往下处理了
		for _, t := range tombstones {
			// Should we skip this block because it contains points that have been deleted
			if t.Min <= block.MinTime && t.Max >= block.MaxTime {
				skip = true
				break
			}
		}

		if skip {
			continue
		}
		
		//TODO: Validate checksum
		temp = temp[:0]
		// The +4 is the 4 byte checksum length
		// 解析当前的datablock, 获取包含的所有Value，每个Value里包括真实值和时间戳两部分
		temp, err = DecodeBlock(m.b[block.Offset+4:block.Offset+int64(block.Size)], temp)
		if err != nil {
			return nil, err
		}

		// Filter out any values that were deleted
		// 再次判断时间戳，看是否已经被标记删除
		for _, t := range tombstones {
			temp = Values(temp).Exclude(t.Min, t.Max)
		}

		values = append(values, temp...)
	}

	return values, nil
}
```

###### indirectIndex
* 读取读析解析tsm文件的Index部分，上面介绍的`mmapAccessor`负责创建它。先来看一下定义:
```
type indirectIndex struct {
	mu sync.RWMutex
		b []byte

	// offsets contains the positions in b for each key.  It points to the 2 byte length of
	// key.
	// 记录了每个子index在tsm文件中的偏移量，便于快速访问某一个子index
	offsets []byte

	// minKey, maxKey are the minium and maximum (lexicographically sorted) contained in the
	// file
	minKey, maxKey []byte

	// minTime, maxTime are the minimum and maximum times contained in the file across all
	// series.
	minTime, maxTime int64

	// tombstones contains only the tombstoned keys with subset of time values deleted.  An
	// entry would exist here if a subset of the points for a key were deleted and the file
	// had not be re-compacted to remove the points on disk.
	
	//记录了某些key对应的已被标记为删除的一系统时间范围
	tombstones map[string][]TimeRange
}
```
* 获取某个子index的文件偏移量
```
func (d *indirectIndex) offset(i int) int {
	if i < 0 || i+4 > len(d.offsets) {
		return -1
	}
	return int(binary.BigEndian.Uint32(d.offsets[i*4 : i*4+4]))
}
```
* 查找某个key对应的子index在tms文件里和偏移量,返回的是indirectIndex.offsets的索引下标，即可获到真正的偏移量。tsm文件的index部分中的key是按从小到大顺序排列的，这个方法中用到的二分查找法只能查找到比这个待查的key下面的所有key中最大的一个
```
func (d *indirectIndex) searchOffset(key []byte) int {
	// 二分法查找
	i := bytesutil.SearchBytesFixed(d.offsets, 4, func(x []byte) bool {
		// i is the position in offsets we are at so get offset it points to
		offset := int32(binary.BigEndian.Uint32(x))

		// It's pointing to the start of the key which is a 2 byte length
		keyLen := int32(binary.BigEndian.Uint16(d.b[offset : offset+2]))

		// See if it matches
		return bytes.Compare(d.b[offset+2:offset+2+keyLen], key) >= 0
	})

	// See if we might have found the right index
	if i < len(d.offsets) {
		return int(i / 4)
	}

	// The key is not in the index.  i is the index where it would be inserted so return
	// a value outside our offset range.
	return int(len(d.offsets)) / 4
}
```
* 获取key对应的所有indexEntry
```
func (d *indirectIndex) ReadEntries(key []byte, entries *[]IndexEntry) []IndexEntry {
	d.mu.RLock()
	defer d.mu.RUnlock()

	ofs := d.search(key)
	if ofs < len(d.b) {
		k, entries := d.readEntriesAt(ofs, entries)
		// The search may have returned an i == 0 which could indicated that the value
		// searched should be inserted at position 0.  Make sure the key in the index
		// matches the search value.
		if !bytes.Equal(key, k) {
			return nil
		}

		return entries
	}

	// The key is not in the index.  i is the index where it would be inserted.
	return nil
}
```
* 从index中获取某些key,其实是重构indirectIndex.offsets,剔除被删除的key对应的文件偏移量
```
func (d *indirectIndex) Delete(keys [][]byte) {
	if len(keys) == 0 {
		return
	}

	if !bytesutil.IsSorted(keys) {
		bytesutil.Sort(keys)
	}

	// Both keys and offsets are sorted.  Walk both in order and skip
	// any keys that exist in both.
	d.mu.Lock()
	start := d.searchOffset(keys[0])
	for i := start * 4; i+4 <= len(d.offsets) && len(keys) > 0; i += 4 {
		offset := binary.BigEndian.Uint32(d.offsets[i : i+4])
		_, indexKey := readKey(d.b[offset:])

		for len(keys) > 0 && bytes.Compare(keys[0], indexKey) < 0 {
			keys = keys[1:]
		}

		if len(keys) > 0 && bytes.Equal(keys[0], indexKey) {
			keys = keys[1:]
			// nilOffset是4个byte, 每个byte是255
			// 将这个位置的offset标识4个255,标识当前key是删除
			copy(d.offsets[i:i+4], nilOffset[:])
		}
	}
	
	//压缩Offset这个数组，剔除其中被标记为4个255的位置
	d.offsets = bytesutil.Pack(d.offsets, 4, 255)
	d.mu.Unlock()
}
```
* 删除某些key在指定的时间戳范围内的index
```
func (d *indirectIndex) DeleteRange(keys [][]byte, minTime, maxTime int64) {
    ...
	
	// 如果给定的时间戳[minTime, maxTime]包括了所有的时间跨度，那就把所有的key都删除
	if minTime == math.MinInt64 && maxTime == math.MaxInt64 {
		d.Delete(keys)
		return
	}

	// 指定的时间戳范围和indirectIndex的时间戳范围没交集，那就没什么可删的
	min, max := d.TimeRange()
	if minTime > max || maxTime < min {
		return
	}

    // fullKeys用来存放key对应的datablock需要全部删除的key的集合
	fullKeys := make([][]byte, 0, len(keys))
	// tombstones用来存放key对应的Value需要部分删除的key对TimeRange列表的
	tombstones := map[string][]TimeRange{}
	var ie []IndexEntry

	for i := 0; len(keys) > 0 && i < d.KeyCount(); i++ {
		k, entries := d.readEntriesAt(d.offset(i), &ie)

		// Skip any keys that don't exist.  These are less than the current key.
		for len(keys) > 0 && bytes.Compare(keys[0], k) < 0 {
			keys = keys[1:]
		}

		// No more keys to delete, we're done.
		if len(keys) == 0 {
			break
		}

		// If the current key is greater than the index one, continue to the next
		// index key.
		if len(keys) > 0 && bytes.Compare(keys[0], k) > 0 {
			continue
		}

		// If multiple tombstones are saved for the same key
		if len(entries) == 0 {
			continue
		}

		// 指定要删除的时间戳范围和当前key的时间戳范围没交集，换下一个
		min, max := entries[0].MinTime, entries[len(entries)-1].MaxTime
		if minTime > max || maxTime < min {
			continue
		}

		// Does the range passed in cover every value for the key?
		// 当前key的时间戳范围是给定的需要删除的时间戳范围的子集，那全部都需要删除
		if minTime <= min && maxTime >= max {
			fullKeys = append(fullKeys, keys[0])
			keys = keys[1:]
			continue
		}

		d.mu.RLock()
		existing := d.tombstones[string(k)]
		d.mu.RUnlock()

		// 部分value在给定的时间戳范围内，需要删除
		newTs := append(existing, append(tombstones[string(k)], TimeRange{minTime, maxTime})...)
		fn := func(i, j int) bool {
			a, b := newTs[i], newTs[j]
			if a.Min == b.Min {
				return a.Max <= b.Max
			}
			return a.Min < b.Min
		}

		// Sort the updated tombstones if necessary
		if len(newTs) > 1 && !sort.SliceIsSorted(newTs, fn) {
			sort.Slice(newTs, fn)
		}

		tombstones[string(k)] = newTs

		minTs, maxTs := newTs[0].Min, newTs[0].Max
		for j := 1; j < len(newTs); j++ {
			prevTs := newTs[j-1]
			ts := newTs[j]

			// Make sure all the tombstone line up for a continuous range.  We don't
			// want to have two small deletes on each edges end up causing us to
			// remove the full key.
			if prevTs.Max != ts.Min-1 && !prevTs.Overlaps(ts.Min, ts.Max) {
				minTs, maxTs = int64(math.MaxInt64), int64(math.MinInt64)
				break
			}

			if ts.Min < minTs {
				minTs = ts.Min
			}
			if ts.Max > maxTs {
				maxTs = ts.Max
			}
		}

		// If we have a fully deleted series, delete it all of it.
		if minTs <= min && maxTs >= max {
			fullKeys = append(fullKeys, keys[0])
			keys = keys[1:]
			continue
		}
	}

	// Delete all the keys that fully deleted in bulk
	if len(fullKeys) > 0 {
		d.Delete(fullKeys)
	}

	if len(tombstones) == 0 {
		return
	}

	d.mu.Lock()
	for k, v := range tombstones {
		d.tombstones[k] = v
	}
	d.mu.Unlock()
}
```

###### Tombstoner
* influxdb中数据删除是标识删除，写入新的logentry,标记key删除，然后在tsm tree作compaction时真正删掉
* 将哪些key在哪些时间范围内的值需要删除的信息记录到形如`/home/lw/data/influxdb/data/my_test_db_2/autogen/59/000000027-000000002.tombstone`的文件中
* tombstone文件格式从源码里看有过v1,v2,v3,v4多个版本
* 添加到Tombsoner中的key的TimeRange对应的值不一定最终是被删除的，必须要commit了才可以，当然也可以通过rollback取消。这是通过写入到一个临时文件*.tombstone.tmp来实现这个类似事务的功能
* 具体实现比较简单，我们这里不累述了。

###### TSMReader
* 封装了具体的读TSM的操作，先看其定义:
```
type TSMReader struct {
	// refs is the count of active references to this reader.
	refs   int64
	refsWG sync.WaitGroup

	madviseWillNeed bool // Hint to the kernel with MADV_WILLNEED.
	mu              sync.RWMutex

	// accessor provides access and decoding of blocks for the reader.
	// 上面介绍过，mmap方式读tsm文件
	accessor blockAccessor

	// index is the index of all blocks.
	// TSMIndex是interface, 这里就是用的上面介绍过的indirectIndex
	index TSMIndex

	// tombstoner ensures tombstoned keys are not available by the index.
	// 在tombstoner中的都是被删除的，读的时候就不读了;
	// indirectIndex根据这个tomstoner作DeleteRagne操作
	tombstoner *Tombstoner

	// size is the size of the file on disk.
	size int64

	// lastModified is the last time this file was modified on disk
	lastModified int64

	// deleteMu limits concurrent deletes
	deleteMu sync.Mutex
}
```
* 创建`TSMReader`
```
func NewTSMReader(f *os.File, options ...tsmReaderOption) (*TSMReader, error) {
	t := &TSMReader{}
	for _, option := range options {
		option(t)
	}

	stat, err := f.Stat()
	if err != nil {
		return nil, err
	}
	
	// 获取tsm文件基本信息
	t.size = stat.Size()
	t.lastModified = stat.ModTime().UnixNano()
	
	// 创建 mmapAccessor,用内存映射方式来访问tsm文件
	t.accessor = &mmapAccessor{
		f:            f,
		mmapWillNeed: t.madviseWillNeed,
	}

    // 读取并解析tsm文件的index部分
	index, err := t.accessor.init()
	if err != nil {
		return nil, err
	}

	t.index = index
	
	// 创建 tombstoner, 从tombstone文件中获取已被标记为删除的key对应的time range信息
	t.tombstoner = NewTombstoner(t.Path(), index.ContainsKey)

    // 根据tombstone文件内容从index中剔除被删除的内容
	if err := t.applyTombstones(); err != nil {
		return nil, err
	}

	return t, nil
}
```
* `TSMReader`提供的大部分方式都是通过`blockAccessor`对tsm文件的访问，这里不累述了。

#### FileStore
* `FileStore`用来管理多个TSM文件，其实也就是管理多个`TSMReader`。我们先来看一下其定义:
```
type FileStore struct {
	mu           sync.RWMutex
	lastModified time.Time //被管理的所有TSM文件最后被修改的时间 
	// Most recently known file stats. If nil then stats will need to be
	// recalculated
	lastFileStats []FileStat // 所有被管理的TSM文件的状态

	currentGeneration int
	dir               string // TSM文件所在的目录

	files           []TSMFile //被管理的所有的TSM文件，即TSMReader
	tsmMMAPWillNeed bool          // If true then the kernel will be advised MMAP_WILLNEED for TSM files.
	openLimiter     limiter.Fixed // limit the number of concurrent opening TSM files.

	logger       *zap.Logger // Logger to be used for important messages
	traceLogger  *zap.Logger // Logger to be used when trace-logging is on.
	traceLogging bool

	stats  *FileStoreStatistics
	purger *purger //TSM文件的延迟删除，有可能在删除时文件处于被使用状态，然后将其加入到这个purger中

	currentTempDirID int

	parseFileName ParseFileNameFunc

	obs tsdb.FileStoreObserver
}
```

* `Open`操作：加载指定目录下所有的TSM文件, 代码较多，但其实逻辑相对简单
```
// Open loads all the TSM files in the configured directory.
func (f *FileStore) Open() error {
    ...
	
	// 获取到目录下所有的.tsm文件
	files, err := filepath.Glob(filepath.Join(f.dir, fmt.Sprintf("*.%s", TSMFileExtension)))
	if err != nil {
		return err
	}

	// struct to hold the result of opening each reader in a goroutine
	type res struct {
		r   *TSMReader
		err error
	}

    // 创建Channel, 用来通知TSM加载的结果
	readerC := make(chan *res)
	
	for i, fn := range files {
		// Keep track of the latest ID
		generation, _, err := f.parseFileName(fn)
		if err != nil {
			return err
		}

		if generation >= f.currentGeneration {
			f.currentGeneration = generation + 1
		}

        // 打开当前的TSM文件
		file, err := os.OpenFile(fn, os.O_RDONLY, 0666)
		if err != nil {
			return fmt.Errorf("error opening file %s: %v", fn, err)
		}

        // 每个TSM文件使用一个单独的goroutine来中载
		go func(idx int, file *os.File) {
		    // 限制同时被加载TSM文件的个数，避免资源的过多占用
			f.openLimiter.Take()
			defer f.openLimiter.Release()

			start := time.Now()
			// 针对当前的TSM文件创建TSMReader，用来读TSM文件 
			df, err := NewTSMReader(file, WithMadviseWillNeed(f.tsmMMAPWillNeed))

            ...
			
			df.WithObserver(f.obs)
			readerC <- &res{r: df}
		}(i, file)
	}

	var lm int64
	for range files {
		res := <-readerC
		if res.err != nil {
			return res.err
		} else if res.r == nil {
			continue
		}
		// 加载成功的TSM文件加入到f.files中
		f.files = append(f.files, res.r)
        ...
	}
	f.lastModified = time.Unix(0, lm).UTC()
	close(readerC)

    // 按文件名排序
	sort.Sort(tsmReaders(f.files))
	atomic.StoreInt64(&f.stats.FileCount, int64(len(f.files)))
	return nil
}
```

* `WalkKeys`: 使用指定函数遍历给定key及其后续key
```
func (f *FileStore) WalkKeys(seek []byte, fn func(key []byte, typ byte) error) error {
    ...

    // 在每个TSM文件里搜索key大小等于seek的所有key
	// 调用的是inderictIndex.searchOffset方法，这个方法如果当前tsm文件里不包含这个seek,那就返回这个tsm文件里最大的一个key, 这有点不合理啊～～～
	ki := newMergeKeyIterator(f.files, seek)
	f.mu.RUnlock()
	for ki.Next() {
		key, typ := ki.Read()
		if err := fn(key, typ); err != nil {
			return err
		}
	}

	return nil
}
```

* `DeleteRange`: 删除在给定时间范围内的一系列key对应的值，实际是记录到tombstone文件中
```
// be used with smaller batches of series keys.
func (f *FileStore) DeleteRange(keys [][]byte, min, max int64) error {
	var batches BatchDeleters
	f.mu.RLock()
	
	//遍历所有的TSMReader, 如果当前文件有需要删除的内容，就生成一个batchdeleter
	for _, f := range f.files {
		if f.OverlapsTimeRange(min, max) {
			batches = append(batches, f.BatchDelete())
		}
	}
	f.mu.RUnlock()

	if len(batches) == 0 {
		return nil
	}

	if err := func() error {
		if err := batches.DeleteRange(keys, min, max); err != nil {
			return err
		}

        // Commit成功后就会写入到tombstone文件中
		return batches.Commit()
	}(); err != nil {
		// Rollback the deletes
		_ = batches.Rollback()
		return err
	}

	f.mu.Lock()
	f.lastModified = time.Now().UTC()
	f.lastFileStats = nil
	f.mu.Unlock()
	return nil
}
```

* `Read`读指定时间戳处的key对应的所有value
```
func (f *FileStore) Read(key []byte, t int64) ([]Value, error) {
	f.mu.RLock()
	defer f.mu.RUnlock()

	for _, f := range f.files {
		// Can this file possibly contain this key and timestamp?
		if !f.Contains(key) {
			continue
		}

		// May have the key and time we are looking for so try to find
		v, err := f.Read(key, t)
		if err != nil {
			return nil, err
		}

        // 遍历所有TSM文件读，已读到一个后，就不再遍历后续的TSM文件 
		if len(v) > 0 {
			return v, nil
		}
	}
	return nil, nil
}
```

* `locations`: 根据给定的key, 时间戳和排序规则（升序，降序）返回`location`列表，主要是在数据查询时会用到
我们先来看下`location`定义：
```
type location struct {
	r     TSMFile  // 属于哪个TSM文件 
	entry IndexEntry //封装的是哪个IndexEntry

	readMin, readMax int64 // 一个IndexEntry包含了一段时间范围，这两个值表示已经读过了这个IndexEntry中的哪段时间的数据
}

 //当前location没有可读的数据了，返回true;反之，返回false
func (l *location) read() bool {
	return l.readMin <= l.entry.MinTime && l.readMax >= l.entry.MaxTime
}
```
我们看下 `locations`的实现:
```
func (f *FileStore) locations(key []byte, t int64, ascending bool) []*location {
	var cache []IndexEntry
	locations := make([]*location, 0, len(f.files))
	for _, fd := range f.files {
		minTime, maxTime := fd.TimeRange()

        // 先根据当前TSM文件覆盖的时间范围过滤
		if ascending && maxTime < t {
			continue
		} else if !ascending && minTime > t {
			continue
		}
		tombstones := fd.TombstoneRange(key)

        // 过滤所有的index entry
		entries := fd.ReadEntries(key, &cache)
	LOOP:
		for i := 0; i < len(entries); i++ {
			ie := entries[i]

            // 过滤掉已经被标记删除的
			for _, t := range tombstones {
				if t.Min <= ie.MinTime && t.Max >= ie.MaxTime {
					continue LOOP
				}
			}

			if ascending && ie.MaxTime < t {
				continue
				// If we descending and the min time of a block is after where we are looking, skip
				// it since the data is out of our range
			} else if !ascending && ie.MinTime > t {
				continue
			}

            // 生成location
			location := &location{
				r:     fd,
				entry: ie,
			}

            // 初始化已读时间戳范围
			if ascending {
				location.readMin = math.MinInt64
				location.readMax = t - 1
			} else {
				location.readMin = t + 1
				location.readMax = math.MaxInt64
			}
			// Otherwise, add this file and block location
			locations = append(locations, location)
		}
	}
	return locations
}
```

#### KeyCursor
按给定的时间戳和排序规则，对所有datablock作排序后输出，每次调用Next后，都会输出一部分排序好的Datablock
```
type KeyCursor struct {
	key []byte

	// seeks is all the file locations that we need to return during iteration.
	seeks []*location

	// current is the set of blocks possibly containing the next set of points.
	// Normally this is just one entry, but there may be multiple if points have
	// been overwritten.
	current []*location
	buf     []Value

	ctx context.Context
	col *metrics.Group

	// pos is the index within seeks.  Based on ascending, it will increment or
	// decrement through the size of seeks slice.
	pos       int
	ascending bool
}
```