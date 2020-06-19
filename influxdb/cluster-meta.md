#### Cluster版本中的Meta 
##### Metadata Client
###### Metadata Client概述
0. 定义在 `services/meta/client.go`中;
1. Cluster 版本中的Meta是本地的一个内存缓存，数据来源MetaServer;
2. 对Meta的所有写操作，也将通过http+pb的方式发送到MetaServer, 然后阻塞等待从MetaServer返回的新的Metadata通知;
3. MetaClient通过http long polling来及时获取Metadata的变化;

###### 所有和Meta data相关的请求
定义在`services/meta/internal/meto.proto`
```
enum Type {
		CreateNodeCommand                = 1;
		DeleteNodeCommand                = 2;
		CreateDatabaseCommand            = 3;
		DropDatabaseCommand              = 4;
		CreateRetentionPolicyCommand     = 5;
		DropRetentionPolicyCommand       = 6;
		SetDefaultRetentionPolicyCommand = 7;
		UpdateRetentionPolicyCommand     = 8;
		CreateShardGroupCommand          = 9;
		DeleteShardGroupCommand          = 10;
		CreateContinuousQueryCommand     = 11;
		DropContinuousQueryCommand       = 12;
		CreateUserCommand                = 13;
		DropUserCommand                  = 14;
		UpdateUserCommand                = 15;
		SetPrivilegeCommand              = 16;
		SetDataCommand                   = 17;
		SetAdminPrivilegeCommand         = 18;
		UpdateNodeCommand                = 19;
		CreateSubscriptionCommand        = 21;
		DropSubscriptionCommand          = 22;
		RemovePeerCommand                = 23;
		CreateMetaNodeCommand            = 24;
		CreateDataNodeCommand            = 25;
		UpdateDataNodeCommand            = 26;
		DeleteMetaNodeCommand            = 27;
		DeleteDataNodeCommand            = 28;
		SetMetaNodeCommand               = 29;
    }

```

###### 重点方法分析
1. `retryUntilExec`: 发送请求到MetadataServer, 直到成功返回或到达最大的重试次数;
```
func (c *Client) retryUntilExec(typ internal.Command_Type, desc *proto.ExtensionDesc, value interface{}) error {
	var err error
	var index uint64
	tries := 0
	currentServer := 0
	var redirectServer string

	for {
		c.mu.RLock()
		// exit if we're closed
		// 如果Client被关闭，我们立即退出
		select {
		case <-c.closing:
			c.mu.RUnlock()
			return nil
		default:
			// we're still open, continue on
		}
		c.mu.RUnlock()

		// build the url to hit the redirect server or the next metaserver
		// 构造请求的Url, 失败时会遍历metaServer发送消息
		var url string
		if redirectServer != "" {
			url = redirectServer
			redirectServer = ""
		} else {
			c.mu.RLock()
			if currentServer >= len(c.metaServers) {
				currentServer = 0
			}
			server := c.metaServers[currentServer]
			c.mu.RUnlock()

			url = fmt.Sprintf("://%s/execute", server)
			if c.tls {
				url = "https" + url
			} else {
				url = "http" + url
			}
		}

        // 发送http请求，成功时返回index，标示当前的metadata版本
		index, err = c.exec(url, typ, desc, value)
		tries++
		currentServer++

		if err == nil {
		    // 等待本地的meta data更新到最新, meta data版本用index来标识
			c.waitForIndex(index)
			return nil
		}

		if tries > maxRetries {
			return err
		}

        ...
		
		time.Sleep(errSleep)
	}
}
```
2. `pollForUpdates`: 通过http请求从MetaServer拉取当前MetaData的snapshot,并通知Metadata有改变
```
	for {
		data := c.retryUntilSnapshot(c.index())
		if data == nil {
			// this will only be nil if the client has been closed,
			// so we can exit out
			return
		}

		// update the data and notify of the change
		c.mu.Lock()
		idx := c.cacheData.Index
		c.cacheData = data
		c.updateAuthCache()
		if idx < data.Index {
		    // 通过chan通过Metadata变化 
			close(c.changed)
			c.changed = make(chan struct{})
		}
		c.mu.Unlock()
	}
```
3. `Client.Open`: 从MetaServer拉取meta snapshot并且开启新的goroutine来拉取Metadata更新
```
func (c *Client) Open() error {
	c.changed = make(chan struct{})
	c.closing = make(chan struct{})
	c.cacheData = c.retryUntilSnapshot(0)

	go c.pollForUpdates()

	return nil
}
```

##### Metadata Server
###### 概述
1. 这是一个CP系统，对metadata采用强一致的存储
2. raft实现：https://github.com/hashicorp/raft
3. 存储: https://github.com/hashicorp/raft-boltdb
4. Meta节点间使用tcp通讯， MetaClient和MetaServer间使用Http通讯

###### MetaService启动
1. 定义在`services/meta/service.go`, Http服务启动
2. Http请求处理在`services/meata/handler.go`中, 如果当前的MetaNode不是leader, http请求重定向到Leader,实现上是把leaer http url返回给请求客户端;

###### Meta请求的执行
1. `Handler.store.apply(body)` 来处理具体的请求，走raft一致性写入流程，将序列化后的command作为log写入，log entry被committed后，apply到状态，然后apply返回
2. Raft相关的操作都定义在`service/meta/store.go`, 在其`open`方法初始化raft相关
```
func (s *store) open(raftln net.Listener) error {
	...

	var initializePeers []string
	if len(joinPeers) > 0 {
	     // 确保其他meta节点的http服务已经open，才继续向下走
		}
	}

	if err := s.setOpen(); err != nil {
		return err
	}

	// Open the raft store.
	// 创建并打开raft store
	if err := s.openRaft(initializePeers, raftln); err != nil {
		return fmt.Errorf("raft: %s", err)
	}

    // 等待leader被选举出来
	if err := s.waitForLeader(0); err != nil {
		return err
	}
	
	...
	
	return nil
}
```
3. command作为log entry被raft给committed后，要apply到fsm, 相应的操作定义在`services/meta/store_fsm.go`中
```
func (fsm *storeFSM) Apply(l *raft.Log) interface{} {
	var cmd internal.Command
	if err := proto.Unmarshal(l.Data, &cmd); err != nil {
		panic(fmt.Errorf("cannot marshal command: %x", l.Data))
	}

	// Lock the store.
	s := (*store)(fsm)
	s.mu.Lock()
	defer s.mu.Unlock()

	err := func() interface{} {
		switch cmd.GetType() {
		case internal.Command_RemovePeerCommand:
	    // 处理各种情况，主要是调用 `services/meta/data.go`中的接口，更改meta信息	
		...
	}()

	// Copy term and index to new metadata.
	fsm.data.Term = l.Term
	fsm.data.Index = l.Index

	// signal that the data changed
	close(s.dataChanged)
	s.dataChanged = make(chan struct{})

	return err
}
```
4. 启动中raft会回调`services/meta/store_fsm.go`中的`Restore`接口，从`snapshot`加载meta信息到store.data
5. 在上面的`Apply`函数中，apply成功后，data.index会被更新，同时会调用`close(s.dataChanged)`，通知这个chan作通知
6. 在上面我们讲过MetaClient通过`pollForUpdates`来及时取回变更后的MetaData,如果当前MetaData没有变更，即Client和Server端的data.Index是相同的，这个请求将在MetaServer端被hold信;有变更后再返回
```
func (h *handler) serveSnapshot(w http.ResponseWriter, r *http.Request) {
	...

	select {
	case <-h.store.afterIndex(index): //等待s.dataChanged的通知，被close后返回
		// Send updated snapshot to client.
		ss, err := h.store.snapshot()
		if err != nil {
			h.httpError(err, w, http.StatusInternalServerError)
			return
		}
		b, err := ss.MarshalBinary()
		if err != nil {
			h.httpError(err, w, http.StatusInternalServerError)
			return
		}
		w.Header().Add("Content-Type", "application/octet-stream")
		w.Write(b)
		return
		
		...
		
	}
}
```
