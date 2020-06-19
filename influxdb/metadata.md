#### Meta分析
##### 一图抵千言
![metadata.png](https://upload-images.jianshu.io/upload_images/2020390-116c2e61882f3b02.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 按上图，我们从右到从逐一简单介绍一下
###### ShardInfo
1. 定义了一个Shard的id和它位于哪个data node上;

###### ShardGroupInfo
1. 封装了ShardGroup的相关信息
2. Influxdb是按时间写入数据的，每个DB都有自己的`Retention Policy`(这个我们后面会介绍)，这个`Retention Policy`规定了每两个ShardGroup之间的时间跨度`ShardGroup Duration`, 即每过一个`ShardGrup Duration`就会生产切换到下一个新的ShardGroup;
3. 这个ShardGroupInfo就记录了当前这个ShardGroup的相关信息，比较主要的信息有：
3.1 StartTime: 这个Group里最早的时间
3.2 EndTime: 这个Group里最晚的时间
3.3 根据上面的两个时间，我们就可以按时间和时间范围来查找到相应的ShardGroup; 
3.4 Shards      []ShardInfo: 这个ShardGroup包含的所有Shard,对于同一个ShardGroup，按Series key(Point key)不同散列写到不同的Shard中;

###### RetentionPolicyInfo
1. 封装了Retention Policy: 包括了复本个数，数据保留时长，ShardGroup切分时长和当前节点的所有`ShardGroup`信息
2. 定义了按时间和时间范围查找相应SahrdGroup的方法

###### DatabaseInfo
1. 管理 `RetentionPolicies` 和 `ContinuousQueries`

###### UserInfo
1. 封装了用户信息：用户名，密码，对db的操作权限

###### 总结
1. 上面介绍的每个对象基础都提供了对其管理的下层metadata信息的增，删，查的方法;

#### Meta Client
##### 定义
1. 定义在`services/meta/client.go`中，负责所有和meta data有关的操作和请求处理
```
type Client struct {
	logger *zap.Logger

	mu        sync.RWMutex
	closing   chan struct{}
	changed   chan struct{}
	cacheData *Data

	// Authentication cache.
	authCache map[string]authUser

	path string

	retentionAutoCreate bool
}
```
主要就是操作上面介绍过的`cacheData *Data`;
2. **提供了大量的方法，基出上都是对上述`Data`类型包含的meta信息的增，删，查，改操作**

##### 主要方法介绍
1. `snapshot`方法：将meta数据写入磁盘，所有的meta信息都有对应的protocol buffer结构，依赖protocol buffer作序列化和反序列化:
```
func snapshot(path string, data *Data) error {
	filename := filepath.Join(path, metaFile)
	tmpFile := filename + "tmp"

	f, err := os.Create(tmpFile)

	defer f.Close()

	var d []byte
	//利用protocol buffer作二进制的序列化
	if b, err := data.MarshalBinary(); err != nil {
		return err
	} else {
		d = b
	}

    //写入文件 
	if _, err := f.Write(d); err != nil {
		return err
	}

	if err = f.Sync(); err != nil {
		return err
	}

	//close file handle before renaming to support Windows
	if err = f.Close(); err != nil {
		return err
	}

	return file.RenameFile(tmpFile, filename)
}
```
2. `Load`方法：meta数据是会保存到磁盘的，influxdb启动时也会从磁盘上读取:
```
func (c *Client) Load() error {
	file := filepath.Join(c.path, metaFile)

	f, err := os.Open(file)
	defer f.Close()

	data, err := ioutil.ReadAll(f)

    //利用protocol buffer作反序列化
	if err := c.cacheData.UnmarshalBinary(data); err != nil {
		return err
	}
	return nil
}
```
3. `commit`方法：influxdb运行时，所有的meta信息在内存里都缓存一分，当meta信息有改动时，通过此方法立即写入磁盘，同时更新内存里的缓存
```
func (c *Client) commit(data *Data) error {
	data.Index++

	// try to write to disk before updating in memory
	if err := snapshot(c.path, data); err != nil {
		return err
	}

	// update in memory
	c.cacheData = data

	// close channels to signal changes
	close(c.changed)
	c.changed = make(chan struct{})

	return nil
}
```
4. `ShardGroupsByTimeRange`和` ShardsByTimeRange`:按给定的时间查找已有的ShardGroup和Shard
```
func (c *Client) ShardGroupsByTimeRange(database, policy string, min, max time.Time) (a []ShardGroupInfo, err error) {
	...
	// 先找到RetentionPolicyInfo
	rpi, err := c.cacheData.RetentionPolicy(database, policy)
	if err != nil {
		return nil, err
	} else if rpi == nil {
		return nil, influxdb.ErrRetentionPolicyNotFound(policy)
	}
	groups := make([]ShardGroupInfo, 0, len(rpi.ShardGroups))
	
	//遍历RPI中的所有ShardGroup
	for _, g := range rpi.ShardGroups {
		if g.Deleted() || !g.Overlaps(min, max) {
			continue
		}
		groups = append(groups, g)
	}
	return groups, nil
}
```
5. `PrecreateShardGroups`: 预先创建ShardGroup, 避免在相应时间段数据到达时才创建ShardGroup
```
func (c *Client) PrecreateShardGroups(from, to time.Time) error {
	c.mu.Lock()
	defer c.mu.Unlock()
	data := c.cacheData.Clone()
	var changed bool

    // 遍历所有的DatabaseInfo信息
	for _, di := range data.Databases {
		for _, rp := range di.RetentionPolicies {
			if len(rp.ShardGroups) == 0 {
				// No data was ever written to this group, or all groups have been deleted.
				continue
			}
			
			// ShardGroups中的所有ShardGroup已经是按时间排序好的，最后一个也就是最新的一个ShardGroup
			g := rp.ShardGroups[len(rp.ShardGroups)-1] // Get the last group in time.
			
			// 
			if !g.Deleted() && g.EndTime.Before(to) && g.EndTime.After(from) {
				// Group is not deleted, will end before the future time, but is still yet to expire.
				// This last check is important, so the system doesn't create shards groups wholly
				// in the past.

				// Create successive shard group.
				// 计算出需要创建的ShardGroup的开始时间
				nextShardGroupTime := g.EndTime.Add(1 * time.Nanosecond)
				// if it already exists, continue
				if sg, _ := data.ShardGroupByTimestamp(di.Name, rp.Name, nextShardGroupTime); sg != nil {
					continue
				}
				newGroup, err := createShardGroup(data, di.Name, rp.Name, nextShardGroupTime)
				if err != nil {
					continue
				}
				changed = true
			}
		}
	}

	if changed {
		if err := c.commit(data); err != nil {
			return err
		}
	}

	return nil
}
```
Influxdb定义了一个Service:Precreator Serivec(services/precreator/service.go)，实现比较简单，周期性的调用`PrecreateShardGroups`,看是否需要创建ShardGroup
```
func (s *Service) runPrecreation() {
	defer s.wg.Done()

	for {
		select {
		case <-time.After(s.checkInterval):
			if err := s.precreate(time.Now().UTC()); err != nil {
				s.Logger.Info("Failed to precreate shards", zap.Error(err))
			}
		case <-s.done:
			s.Logger.Info("Terminating precreation service")
			return
		}
	}
}
```