[toc]

#### Cluster下的数据写入

##### 数据写入的实现
1. 主要分析`cluster/points_writer.go`中的`WritePoints`函数的实现
```
// WritePoints writes across multiple local and remote data nodes according the consistency level.
func (w *PointsWriter) WritePoints(p *WritePointsRequest) error {
	w.statMap.Add(statWriteReq, 1)
	w.statMap.Add(statPointWriteReq, int64(len(p.Points)))

    //2.1 先获取RetentionPolicy
	if p.RetentionPolicy == "" {
		db, err := w.MetaClient.Database(p.Database)
		if err != nil {
			return err
		} else if db == nil {
			return influxdb.ErrDatabaseNotFound(p.Database)
		}
		p.RetentionPolicy = db.DefaultRetentionPolicy
	}

    // 2.2 生成 shardMap
	shardMappings, err := w.MapShards(p)
	if err != nil {
		return err
	}

	// Write each shard in it's own goroutine and return as soon
	// as one fails.
	ch := make(chan error, len(shardMappings.Points))
	for shardID, points := range shardMappings.Points {
	
	    // 2.3 写入数据到Shard
		go func(shard *meta.ShardInfo, database, retentionPolicy string, points []models.Point) {
			ch <- w.writeToShard(shard, p.Database, p.RetentionPolicy, p.ConsistencyLevel, points)
		}(shardMappings.Shards[shardID], p.Database, p.RetentionPolicy, points)
	}

	// Send points to subscriptions if possible.
	ok := false
	// We need to lock just in case the channel is about to be nil'ed
	w.mu.RLock()
	select {
	case w.subPoints <- p:
		ok = true
	default:
	}
	w.mu.RUnlock()
	if ok {
		w.statMap.Add(statSubWriteOK, 1)
	} else {
		w.statMap.Add(statSubWriteDrop, 1)
	}

    // 2.4 等待写入完成 
	for range shardMappings.Points {
		select {
		case <-w.closing:
			return ErrWriteFailed
		case err := <-ch:
			if err != nil {
				return err
			}
		}
	}
	return nil
}
```
2. 上面的函数实现主要分如下几个步骤
2.1 获取对应的RetentionPolicy
2.2 生成ShardMap, 将各个point对应到相应ShardGroup中的Shard中, 这步很关键
2.3 按ShardId不同，开启新的goroutine, 将points写入相应的Shard,可能设计对写入数据到其它的DataNode上;
2.4 等待写入完成或退出

##### ShardMap的生成
1. 先讲一下ShardGroup的概念
1.1 写入Influxdb的每一条数据对带有相应的time时间，每一个SharGroup都有自己的start和end时间，这个时间跨度是由用户写入时选取的RetentionPolicy时的ShardGroupDarution决定，这样每条写入的数据就必然仅属于一个确定的ShardGroup中;

2. 主要实现在`cluster/points_writer.go`中的`MapShards`中
```
func (w *PointsWriter) MapShards(wp *WritePointsRequest) (*ShardMapping, error) {

	// holds the start time ranges for required shard groups
	timeRanges := map[time.Time]*meta.ShardGroupInfo{}

	rp, err := w.MetaClient.RetentionPolicy(wp.Database, wp.RetentionPolicy)
	if err != nil {
		return nil, err
	}
	if rp == nil {
		return nil, influxdb.ErrRetentionPolicyNotFound(wp.RetentionPolicy)
	}

	for _, p := range wp.Points {
		timeRanges[p.Time().Truncate(rp.ShardGroupDuration)] = nil
	}

	// holds all the shard groups and shards that are required for writes
	for t := range timeRanges {
		sg, err := w.MetaClient.CreateShardGroup(wp.Database, wp.RetentionPolicy, t)
		if err != nil {
			return nil, err
		}
		timeRanges[t] = sg
	}

	mapping := NewShardMapping()
	for _, p := range wp.Points {
		sg := timeRanges[p.Time().Truncate(rp.ShardGroupDuration)]
		sh := sg.ShardFor(p.HashID())
		mapping.MapPoint(&sh, p)
	}
	return mapping, nil
}
```
3. 我们来拆解下上面函数的实现
3.1 扫描所有的points, 按时间确定我们需要多个ShardGroup
```
for _, p := range wp.Points {
		timeRanges[p.Time().Truncate(rp.ShardGroupDuration)] = nil
	}
```
3.2 调用`w.MetaClient.CreateShardGroup`, 如果ShardGroup存在直接返回ShardGroup信息，如果不存在创建，创建过程涉及到将CreateShardGroup的请求发送给MetadataServer并等待本地更新到新的MetaData数据;
```
sg, err := w.MetaClient.CreateShardGroup(wp.Database, wp.RetentionPolicy, t)
```
3.3 分析ShardGroup的分配规则， 在`services/meta/data.go`中的`CreateShardGroup`
```
func (data *Data) CreateShardGroup(database, policy string, timestamp time.Time) error {
    ...

	// Require at least one replica but no more replicas than nodes.
	// 确认复本数，不能大于DataNode节点总数
	replicaN := rpi.ReplicaN
	if replicaN == 0 {
		replicaN = 1
	} else if replicaN > len(data.DataNodes) {
		replicaN = len(data.DataNodes)
	}

	// Determine shard count by node count divided by replication factor.
	// This will ensure nodes will get distributed across nodes evenly and
	// replicated the correct number of times.
	// 根据复本数确定Shard数量
	shardN := len(data.DataNodes) / replicaN

	// Create the shard group.
	// 创建ShardGroup
	data.MaxShardGroupID++
	sgi := ShardGroupInfo{}
	sgi.ID = data.MaxShardGroupID
	sgi.StartTime = timestamp.Truncate(rpi.ShardGroupDuration).UTC()
	sgi.EndTime = sgi.StartTime.Add(rpi.ShardGroupDuration).UTC()

	// Create shards on the group.
	sgi.Shards = make([]ShardInfo, shardN)
	for i := range sgi.Shards {
		data.MaxShardID++
		sgi.Shards[i] = ShardInfo{ID: data.MaxShardID}
	}

	// Assign data nodes to shards via round robin.
	// Start from a repeatably "random" place in the node list.
	// ShardInfo中的Owners记录了当前Shard所有复本所在DataNode的信息
	// 分Shard的所有复本分配DataNode
	// 使用data.Index作为基数确定开始的DataNode,然后使用 round robin策略分配
	// data.Index:每次meta信息有更新，Index就会更新, 可以理解为meta信息的版本号
	nodeIndex := int(data.Index % uint64(len(data.DataNodes)))
	for i := range sgi.Shards {
		si := &sgi.Shards[i]
		for j := 0; j < replicaN; j++ {
			nodeID := data.DataNodes[nodeIndex%len(data.DataNodes)].ID
			si.Owners = append(si.Owners, ShardOwner{NodeID: nodeID})
			nodeIndex++
		}
	}

	// Retention policy has a new shard group, so update the policy. Shard
	// Groups must be stored in sorted order, as other parts of the system
	// assume this to be the case.
	rpi.ShardGroups = append(rpi.ShardGroups, sgi)
	sort.Sort(ShardGroupInfos(rpi.ShardGroups))

	return nil
}
```
3.3 按每一个具体的point对应到ShardGroup中的一个Shard: 按point的HashID来对Shard总数取模，HashID是`measurment + tag set`的Hash值
```
for _, p := range wp.Points {
		sg := timeRanges[p.Time().Truncate(rp.ShardGroupDuration)]
		sh := sg.ShardFor(p.HashID())
		mapping.MapPoint(&sh, p)
	}
....

 func (sgi *ShardGroupInfo) ShardFor(hash uint64) ShardInfo {
	return sgi.Shards[hash%uint64(len(sgi.Shards))]
}
```

##### 数据按一致性要求写入
1. 过程简述
1.1 根据一致性要求确认需要成功写入几份
```
switch consistency {
    // 对于ConsistencyLevelAny, ConsistencyLevelOne只需要写入一份即满足一致性要求，返回客户端
	case ConsistencyLevelAny, ConsistencyLevelOne:
		required = 1
	case ConsistencyLevelQuorum:
		required = required/2 + 1
	}
```
1.2 根据Shard.Owners对应的DataNode, 向其中的每个DataNode写入数据，如果是本机，直接调用` w.TSDBStore.WriteToShard`写入;如果非本机，调用`err := w.ShardWriter.WriteShard(shardID, owner.NodeID, points)`;
1.3 写入远端失败时，数据写入HintedHandoff本地磁盘队列多次重试写到远端，直到数据过期被清理;对于一致性要求是`ConsistencyLevelAny`, 写入本地HintedHandoff成功，就算是写入成功;
```
	w.statMap.Add(statWritePointReqHH, int64(len(points)))
				hherr := w.HintedHandoff.WriteShard(shardID, owner.NodeID, points)
				if hherr != nil {
					ch <- &AsyncWriteResult{owner, hherr}
					return
				}

				if hherr == nil && consistency == ConsistencyLevelAny {
					ch <- &AsyncWriteResult{owner, nil}
					return
			}
```
1.4 等待写入超时或完成
```
for range shard.Owners {
		select {
		case <-w.closing:
			return ErrWriteFailed
		case <-timeout:
			w.statMap.Add(statWriteTimeout, 1)
			// return timeout error to caller
			return ErrTimeout
		case result := <-ch:
			// If the write returned an error, continue to the next response
			if result.Err != nil {
				if writeError == nil {
					writeError = result.Err
				}
				continue
			}

			wrote++

			// 写入已达到一致性要求，就立即返回
			if wrote >= required {
				w.statMap.Add(statWriteOK, 1)
				return nil
			}
		}
	}
```

##### HintedHandoff服务
1. 定义在`services/hh/service.go`中
2. 写入HintedHandoff中的数据，按NodeID的不同写入不同的目录，每个目录下又分多个文件，每个文件作为一个segment, 命名规则就是依次递增的id, id的大小按序就是写入的时间按从旧到新排序;
![hitnedhandoff](file:///home/lw/docs/influxdb/hitnedhandoff.png)
3. HintedHandoff服务会针对每一个远端DataNode创建`NodeProcessor`, 每个负责自己DataNode的写入, 运行在一个单独的goroutine中
4. 在每个goroutine中，作两件事：一个是定时清理过期的数据，如果被清理掉的数据还没有成功写入到远端，则会丢失;二是从文件读取数据写入到远端;
```
func (n *NodeProcessor) run() {
	defer n.wg.Done()

	...

	for {
		select {
		case <-n.done:
			return

		case <-time.After(n.PurgeInterval):
			if err := n.queue.PurgeOlderThan(time.Now().Add(-n.MaxAge)); err != nil {
				n.Logger.Printf("failed to purge for node %d: %s", n.nodeID, err.Error())
			}

		case <-time.After(currInterval):
			limiter := NewRateLimiter(n.RetryRateLimit)
			for {
				c, err := n.SendWrite()
				if err != nil {
					if err == io.EOF {
						// No more data, return to configured interval
						currInterval = time.Duration(n.RetryInterval)
					} else {
						currInterval = currInterval * 2
						if currInterval > time.Duration(n.RetryMaxInterval) {
							currInterval = time.Duration(n.RetryMaxInterval)
						}
					}
					break
				}

				// Success! Ensure backoff is cancelled.
				currInterval = time.Duration(n.RetryInterval)

				// Update how many bytes we've sent
				limiter.Update(c)

				// Block to maintain the throughput rate
				time.Sleep(limiter.Delay())
			}
		}
	}
}
```
5. 数据的本地存储和读取
5.1 定义在`services/hh/queue.go`,所有的segment file在内存中组织成一个队列，读从head指向的segment读取，写入到tail指向的segment, 每个segment文件的最后8字节记录当前segment文件已经读到什么位置
5.2 清理，当这个segment文件内容都发送完当前文件会被删除，周期性清理每次只会check当前head指向的segment是否需要清理掉
