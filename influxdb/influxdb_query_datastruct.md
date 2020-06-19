[toc]

#### 前言
* 这里强烈建议先熟悉influxsql的查询语句，可参考 [Data exploration using InfluxQL](https://docs.influxdata.com/influxdb/v1.7/query_language/data_exploration)

#### 关于Select查询请求结果涉及到的一些数据结构
##### Series
1. 定义
```
type Series struct {
	// Name is the measurement name.
	Name string

	// Tags for the series.
	Tags Tags

	id uint64
}
type Tags struct {
	id string
	m  map[string]string
}
```
2. `Series`其实就是`measurement`和`tags`的组合，`tags`是tag key和tag value的map.这个Tags的id是如何产生的呢，其实就是对tag key和tag value编码到`[]byte`: `tagkey1\0tagkey2\0...\tagvalue1\0tagvalue2\0...`,具体实现定义在`query/point.go`中的`encodeTags`

##### Row
1. 定义
```
type Row struct {
	Time int64

	// Series contains the series metadata for this row.
	Series Series

	// Values contains the values within the current row.
	Values []interface{}
}
```
2. `Row`表示查询结果集中的每一行, 其中的`Values`表示是返回的Fields的集合

##### Iterator
###### bufFloatIterator
1. 定义
```
type bufFloatIterator struct {
	itr FloatIterator
	buf *FloatPoint
}
```
相当于c里面的链表元素，`itr`指向下一个元素的指针，`buf`表示当前元素，即`FloatPoint`类型的链表的迭代器.
2. 看一下`FloatPoint`定义
```
type FloatPoint struct {
	Name string
	Tags Tags

	Time  int64
	Value float64
	Aux   []interface{}

	// Total number of points that were combined into this point from an aggregate.
	// If this is zero, the point is not the result of an aggregate function.
	Aggregated uint32
	Nil        bool
}
```
定义在`query/point.gen.go`中， 表示一条field为float类型的数据

3. `Next`实现
```
func (itr *bufFloatIterator) Next() (*FloatPoint, error) {
	buf := itr.buf
	if buf != nil {
		itr.buf = nil
		return buf, nil
	}
	return itr.itr.Next()
}
```
当前`Iterator`的值不为空，就返回当前的`buf`, 当前的值为空，就返回`itr.itr.Next()`,即指向的下一个元素
4. `unread`: iterator回退操作
```
func (itr *bufFloatIterator) unread(v *FloatPoint) { itr.buf = v }
```

###### `floatMergeIterator`
1. 组合了多个`floatIterator`
2. 我们来看下定义
```
type floatMergeIterator struct {
	inputs []FloatIterator
	heap   *floatMergeHeap
	init   bool

	closed bool
	mu     sync.RWMutex

	// Current iterator and window.
	curr   *floatMergeHeapItem
	window struct {
		name      string
		tags      string
		startTime int64
		endTime   int64
	}
}
```
因为要作merge, 这里需要对其管理的所有Interator元素作排序，这里用到了golang的[container/heap](https://golang.org/pkg/container/heap/)作堆排, 大小根堆不太清楚了大家自行google吧。
因为要用`golang的container/heap`来管理，需要实现下面规定的接口，
```
type Interface interface {
	sort.Interface
	Push(x interface{}) // add x as element Len()
	Pop() interface{}   // remove and return element Len() - 1.
}
```
`floatMergeIterator`定义中的`floatMergeHeap`即实现了上面的接口,我们主要来看一下比较函数的实现，比较的其实就是`FloatPoint`
```
func (h *floatMergeHeap) Less(i, j int) bool {
	x, err := h.items[i].itr.peek()
	if err != nil {
		return true
	}
	y, err := h.items[j].itr.peek()
	if err != nil {
		return false
	}

	if h.opt.Ascending {
		if x.Name != y.Name {
			return x.Name < y.Name
		} else if xTags, yTags := x.Tags.Subset(h.opt.Dimensions), y.Tags.Subset(h.opt.Dimensions); xTags.ID() != yTags.ID() {
			return xTags.ID() < yTags.ID()
		}
	} else {
		if x.Name != y.Name {
			return x.Name > y.Name
		} else if xTags, yTags := x.Tags.Subset(h.opt.Dimensions), y.Tags.Subset(h.opt.Dimensions); xTags.ID() != yTags.ID() {
			return xTags.ID() > yTags.ID()
		}
	}

	xt, _ := h.opt.Window(x.Time)
	yt, _ := h.opt.Window(y.Time)

	if h.opt.Ascending {
		return xt < yt
	}
	return xt > yt
}
```
比较的优先级先是`FloatPoint`的measurement名，然后是tagset id, 最后是time,将这个比较函数我们就可以知道.
3. 看一下结构：
![float_merge_iterator](file:///home/lw/docs/influxdb/float_merge_iterator.png)
4. `Next`函数的实现
```
func (itr *floatMergeIterator) Next() (*FloatPoint, error) {
	itr.mu.RLock()
	defer itr.mu.RUnlock()
	if itr.closed {
		return nil, nil
	}
 
    // 堆排的heap数据结构并不会一开始就构造，而是在首次遍历时初始化构造 
	if !itr.init {
		items := itr.heap.items
		itr.heap.items = make([]*floatMergeHeapItem, 0, len(items))
		for _, item := range items {
			if p, err := item.itr.peek(); err != nil {
				return nil, err
			} else if p == nil {
				continue
			}
			itr.heap.items = append(itr.heap.items, item)
		}
		heap.Init(itr.heap)
		itr.init = true
	}

	for {
		// Retrieve the next iterator if we don't have one.
		if itr.curr == nil {
			if len(itr.heap.items) == 0 {
				return nil, nil
			}
			itr.curr = heap.Pop(itr.heap).(*floatMergeHeapItem)

			// Read point and set current window.
			p, err := itr.curr.itr.Next()
			if err != nil {
				return nil, err
			}
			tags := p.Tags.Subset(itr.heap.opt.Dimensions)
			itr.window.name, itr.window.tags = p.Name, tags.ID()
			itr.window.startTime, itr.window.endTime = itr.heap.opt.Window(p.Time)
			return p, nil
		}

		// Read the next point from the current iterator.
		p, err := itr.curr.itr.Next()
		if err != nil {
			return nil, err
		}

		// If there are no more points then remove iterator from heap and find next.
		if p == nil {
			itr.curr = nil
			continue
		}

        // 下面的代码确认是否需要切换Window，
		// Check if the point is inside of our current window.
		inWindow := true
		if window := itr.window; window.name != p.Name {
			inWindow = false
		} else if tags := p.Tags.Subset(itr.heap.opt.Dimensions); window.tags != tags.ID() {
			inWindow = false
		} else if opt := itr.heap.opt; opt.Ascending && p.Time >= window.endTime {
			inWindow = false
		} else if !opt.Ascending && p.Time < window.startTime {
			inWindow = false
		}

		// If it's outside our window then push iterator back on the heap and find new iterator.
		if !inWindow {
			itr.curr.itr.unread(p)
			heap.Push(itr.heap, itr.curr)
			itr.curr = nil
			continue
		}

		return p, nil
	}
}
```
结合上面的`Less`函数可知，针对所有的FloatPoint, 排序的最小单位是Window(由measurement name, tagset id, time window组成)，属性同一Window的FloatPoint不再排序。如果是按升级规则遍历，则遍历的结果是按Window从小到大排，但同一Window内部的多条Point,时间不一定是从小到大的。

###### floatSortedMergeIterator
1. 定义：
```
type floatSortedMergeIterator struct {
	inputs []FloatIterator
	heap   *floatSortedMergeHeap
	init   bool
}
type floatSortedMergeHeap struct {
	opt   IteratorOptions
	items []*floatSortedMergeHeapItem
}
type floatSortedMergeHeapItem struct {
	point *FloatPoint // 这个point用来排序当前所有的Item
	err   error
	itr   FloatIterator
}
```
同样它也借助了`golang/container`中的`heap`， 与`floatMergeIterator`相比它实现了全体`Point`的排序遍历,我们来看一下是如何实现的;
2. `pop`函数:
```
func (itr *floatSortedMergeIterator) pop() (*FloatPoint, error) {
	// Initialize the heap. See the MergeIterator to see why this has to be done lazily.
	if !itr.init {
		items := itr.heap.items
		itr.heap.items = make([]*floatSortedMergeHeapItem, 0, len(items))
		for _, item := range items {
			var err error
			//  在这里为每个 floatSortedMergeHeapItem的point赋值
			if item.point, err = item.itr.Next(); err != nil {
				return nil, err
			} else if item.point == nil {
				continue
			}
			itr.heap.items = append(itr.heap.items, item)
		}
		heap.Init(itr.heap)
		itr.init = true
	}

	if len(itr.heap.items) == 0 {
		return nil, nil
	}

	// Read the next item from the heap.
	item := heap.Pop(itr.heap).(*floatSortedMergeHeapItem)
	if item.err != nil {
		return nil, item.err
	} else if item.point == nil {
		return nil, nil
	}

	// Copy the point for return.
	p := item.point.Clone()

	// 关键点在这里： 在遍历 FloatIterator时候，不是直接遍历，这里重置Item的point,然后重新push这个item, 作堆排，如巧妙~~~
	if item.point, item.err = item.itr.Next(); item.point != nil {
		heap.Push(itr.heap, item)
	}

	return p, nil
}
```

###### floatIteratorScanner
1. 将`floatIterator`的值扫描到map里 
2. 定义
```
type floatIteratorScanner struct {
	input        *bufFloatIterator
	err          error
	keys         []influxql.VarRef
	defaultValue interface{}
}
```
3. `ScanAt`: 在floatIterator中找满足条件的Point, 条件是ts, name, tags均相等,实现比较简单
```
func (s *floatIteratorScanner) ScanAt(ts int64, name string, tags Tags, m map[string]interface{}) {
	if s.err != nil {
		return
	}

    // 获取当前的FloatPoint
	p, err := s.input.Next()
	if err != nil {
		s.err = err
		return
	} else if p == nil {
		s.useDefaults(m)
		return
	} else if p.Time != ts || p.Name != name || !p.Tags.Equals(&tags) {
		s.useDefaults(m)
		// 如果
		s.input.unread(p)
		return
	}

	if k := s.keys[0]; k.Val != "" {
		if p.Nil {
			if s.defaultValue != SkipDefault {
				m[k.Val] = castToType(s.defaultValue, k.Type)
			}
		} else {
			m[k.Val] = p.Value
		}
	}
	for i, v := range p.Aux {
		k := s.keys[i+1]
		switch v.(type) {
		case float64, int64, uint64, string, bool:
			m[k.Val] = v
		default:
			// Insert the fill value if one was specified.
			if s.defaultValue != SkipDefault {
				m[k.Val] = castToType(s.defaultValue, k.Type)
			}
		}
	}
}
```

###### floatParallelIterator
1. 定义：
```
type floatParallelIterator struct {
	input FloatIterator
	ch    chan floatPointError

	once    sync.Once
	closing chan struct{}
	wg      sync.WaitGroup
}
```

2. 在一个单独的goroutine里面循环调用`floatIterator.Next`获取`FloatPoint`,然后写入到chan中：
```
func (itr *floatParallelIterator) monitor() {
	defer close(itr.ch)
	defer itr.wg.Done()

	for {
		// Read next point.
		p, err := itr.input.Next()
		if p != nil {
			p = p.Clone()
		}

		select {
		case <-itr.closing:
			return
		case itr.ch <- floatPointError{point: p, err: err}: //写入数据到Chan
		}
	}
}
```

3. 使用的时候，调用`Next`, 从上面的Chan中读数据:
```
func (itr *floatParallelIterator) Next() (*FloatPoint, error) {
	v, ok := <-itr.ch
	if !ok {
		return nil, io.EOF
	}
	return v.point, v.err
}
```

###### floatLimitIterator
1. 限制在每个window中读取的Point个数
2. 定义：
```
type floatLimitIterator struct {
	input FloatIterator
	opt   IteratorOptions
	n     int

    // 定义了当前的window, measurement和tagset id相同的point算作同一个window group
	prev struct {
		name string
		tags Tags
	}
}
```
3. `Next`:
```
func (itr *floatLimitIterator) Next() (*FloatPoint, error) {
	for {
	    //读取当前的Point
		p, err := itr.input.Next()
		if p == nil || err != nil {
			return nil, err
		}

		// Reset window and counter if a new window is encountered.
		// 判断是否需要切换到新的window, 需要重置计数器n
		if p.Name != itr.prev.name || !p.Tags.Equals(&itr.prev.tags) {
			itr.prev.name = p.Name
			itr.prev.tags = p.Tags
			itr.n = 0
		}

		// Increment counter.
		itr.n++

		// Read next point if not beyond the offset.
		// 首先要跳到开始读取的偏移量Offset
		if itr.n <= itr.opt.Offset {
			continue
		}

		// Read next point if we're beyond the limit.
		// 判断是否已到达Limit限制
		if itr.opt.Limit > 0 && (itr.n-itr.opt.Offset) > itr.opt.Limit {
			continue
		}

		return p, nil
	}
}
```