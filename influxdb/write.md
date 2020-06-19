#### 数据写入流程分析	
1. 本篇不涉及存储层的写入，只分析写入请求的处理流程
##### Influxdb名词介绍
1. 如果想搞清楚Influxdb数据写入流程，Influxdb本身的用法和其一些主要的专用词还是要明白是什么意思，比如measurement, field key,field value, tag key, tag value, tag set, line protocol, point, series, query, retention policy等;
2. 相关的专用名词解释可参考： [InfluxDB glossary of terms](https://docs.influxdata.com/influxdb/v1.6/concepts/glossary/)

##### 分析入口
1. 我们还是以http写请求为入口来分析，在`httpd/handler.go`中创建Handler时有如下代码:
```
	Route{
			"write", // Data-ingest route.
			"POST", "/write", true, writeLogEnabled, h.serveWrite,
		}
```
因此对写入请求的处理就在函数 `func (h *Handler) serveWrite(w http.ResponseWriter, r *http.Request, user meta.User) 
`中。
2. `Handler.serveWrite`流程梳理:
2.1 获取写入的db并判断db是否存在
```
database := r.URL.Query().Get("db")
	if database == "" {
		h.httpError(w, "database is required", http.StatusBadRequest)
		return
	}

	if di := h.MetaClient.Database(database); di == nil {
		h.httpError(w, fmt.Sprintf("database not found: %q", database), http.StatusNotFound)
		return
	}
```
2.2 权限验证
```
if h.Config.AuthEnabled {
		if user == nil {
			h.httpError(w, fmt.Sprintf("user is required to write to database %q", database), http.StatusForbidden)
			return
		}

		if err := h.WriteAuthorizer.AuthorizeWrite(user.ID(), database); err != nil {
			h.httpError(w, fmt.Sprintf("%q user is not authorized to write to database %q", user.ID(), database), http.StatusForbidden)
			return
		}
	}
```
2.3 获取http请求的body部分，如需gzip解压缩则解压，并且作body size的校验，因为有body size大小限制
```
    body := r.Body
	if h.Config.MaxBodySize > 0 {
		body = truncateReader(body, int64(h.Config.MaxBodySize))
	}
	...
	_, err := buf.ReadFrom(body)
```
2.4 从http body中解析出 points
```
points, parseError := models.ParsePointsWithPrecision(buf.Bytes(), time.Now().UTC(),
                       r.URL.Query().Get("precision"))
```
2.5 将解析出的points写入db
```
h.PointsWriter.WritePoints(database, r.URL.Query().Get("rp"), consistency, user, points); 
```

##### Points的解析
1. 将http body解析成Points是写入前的最主要的一步, 相关内容定义在 `models/points.go`中;
2. 我们先来看一下一条写入语句是什么样子的： `insert  test_mea_1,tag1=v1,tag2=v2 cpu=1,memory=10`
其中`test_mea_1`是measurement, tag key是tag1和tag2, 对应的tag value是v1和v2, field key是cpu和memory, field value是1和10;
3. 先来看下`point`的定义，它实现了`Point interface`
```
type point struct {
	time time.Time

    //这个 key包括了measurement和tag set, 且tag set是排序好的	
	key []byte

	// text encoding of field data
	fields []byte

	// text encoding of timestamp
	ts []byte

	// cached version of parsed fields from data
	cachedFields map[string]interface{}

	// cached version of parsed name from key
	cachedName string

	// cached version of parsed tags
	cachedTags Tags

    //用来遍历所有的field
	it fieldIterator
}
```
4. 解析出Points
```
func ParsePointsWithPrecision(buf []byte, defaultTime time.Time, precision string) ([]Point, error) {
	points := make([]Point, 0, bytes.Count(buf, []byte{'\n'})+1)
	var (
		pos    int
		block  []byte
		failed []string
	)
	for pos < len(buf) {
		pos, block = scanLine(buf, pos)
		pos++
  
        ...

		pt, err := parsePoint(block[start:], defaultTime, precision)
		if err != nil {
			failed = append(failed, fmt.Sprintf("unable to parse '%s': %v", string(block[start:]), err))
		} else {
			points = append(points, pt)
		}

	}

	return points, nil
}
```
这里的解析并没有用正则之类的方案，纯的字符串逐次扫描,这里不详细展开说了.

##### PointsWriter分析
1. 定义在`coordinator/points_writer.go`中
2. 主要负责将数据写入到本地的存储,我们重点分析下`WritePointsPrivileged`
```
func (w *PointsWriter) WritePointsPrivileged(database, retentionPolicy string, consistencyLevel models.ConsistencyLevel, points []models.Point) error {
	....
	
	//将point按time对应到相应的Shar上, 这个对应关系存储在shardMappings里, 这个MapShareds我们后面会分析
	shardMappings, err := w.MapShards(&WritePointsRequest{Database: database, RetentionPolicy: retentionPolicy, Points: points})
	if err != nil {
		return err
	}

	// Write each shard in it's own goroutine and return as soon as one fails.
	ch := make(chan error, len(shardMappings.Points))
	for shardID, points := range shardMappings.Points {
	
	    // 每个 Shard启动一个goroutine作写入操作, 真正的写入操作w.writeToShard
		go func(shard *meta.ShardInfo, database, retentionPolicy string, points []models.Point) {
			err := w.writeToShard(shard, database, retentionPolicy, points)
			if err == tsdb.ErrShardDeletion {
				err = tsdb.PartialWriteError{Reason: fmt.Sprintf("shard %d is pending deletion", shard.ID), Dropped: len(points)}
			}
			ch <- err
		}(shardMappings.Shards[shardID], database, retentionPolicy, points)
	}
    ...
	
	// 写入超时会return ErrTimeout
	timeout := time.NewTimer(w.WriteTimeout)
	defer timeout.Stop()
	for range shardMappings.Points {
		select {
		case <-w.closing:
			return ErrWriteFailed
		case <-timeout.C:
			atomic.AddInt64(&w.stats.WriteTimeout, 1)
			// return timeout error to caller
			return ErrTimeout
		case err := <-ch:
			if err != nil {
				return err
			}
		}
	}
	return err
}
```
3. Point到Shard的映谢
3.1 先根据point的time找到对应的ShardGroup, 没有就创建新的ShardGroup;
3.2 按Point的key(measurement + tag set取hash)来散
```
sgi.Shards[hash%uint64(len(sgi.Shards))]
```