## Http请求的处理流程
#### HTTPDService服务的添加
1. 在 Server的启动过程中会添加并启动各种service， 其中就包括这个HTTPDService：```appendHTTPDService(c httpd.Config)``` 定义在 ```cmd/influxdb/run/server.go```中
```
    srv := httpd.NewService(c)
	srv.Handler.MetaClient = s.MetaClient
	srv.Handler.QueryAuthorizer = meta.NewQueryAuthorizer(s.MetaClient)
	srv.Handler.WriteAuthorizer = meta.NewWriteAuthorizer(s.MetaClient)
	srv.Handler.QueryExecutor = s.QueryExecutor
	srv.Handler.Monitor = s.Monitor
	srv.Handler.PointsWriter = s.PointsWriter
	srv.Handler.Version = s.buildInfo.Version
	srv.Handler.BuildType = "OSS"
	ss := storage.NewStore(s.TSDBStore, s.MetaClient)
	srv.Handler.Store = ss
	srv.Handler.Controller = control.NewController(ss, s.Logger)

	s.Services = append(s.Services, srv)
```
2. 从上面的代码可以看出，主要是初始化这个```Handler```, 这个Handler类负责处理具体的Http Request,生成相应的Response;

#### HTTPDService分析
1. Httpd Service的具体实现在 ```services/httpd```目录下
2. 这个http服务使用golang提供的```net/http```包实现
3. 流程解析：
3.1 创建Service:
```
    func NewService(c Config) *Service {
	s := &Service{
		addr:           c.BindAddress, //http服务监控的地址，端口
		https:          c.HTTPSEnabled,
		cert:           c.HTTPSCertificate,
		key:            c.HTTPSPrivateKey,
		limit:          c.MaxConnectionLimit,
		tlsConfig:      c.TLS,
		err:            make(chan error),
		unixSocket:     c.UnixSocketEnabled,
		unixSocketPerm: uint32(c.UnixSocketPermissions),
		bindSocket:     c.BindSocket,
		Handler:        NewHandler(c),  // 创建Handler
		Logger:         zap.NewNop(),
	}
	if s.tlsConfig == nil {
		s.tlsConfig = new(tls.Config)
	}
```
3.2 启动Service:
```
func (s *Service) Open() error {
	s.Handler.Open() // Handler必要的初始化，主要是日志文件的设置

	// Open listener.
	if s.https {
         ...
		//tls listener支持
		s.ln = listener
	} else {
        ...
		listener, err := net.Listen("tcp", s.addr)
		s.ln = listener
	}

	// Open unix socket listener.
	if s.unixSocket {
        ...
		s.unixSocketListener = listener
		go s.serveUnixSocket()
	}

	// Enforce a connection limit if one has been given.
    // 使用这个LimitListener，同时仅能接收s.limit个连接，超过的connect则自动被close掉
	if s.limit > 0 {
		s.ln = LimitListener(s.ln, s.limit)
	}

    ...

	// Begin listening for requests in a separate goroutine.
	go s.serveTCP()
	return nil
}
```
3.3 关键函数之`NewHandler()`:
```
h := &Handler{
		mux:            pat.New(),
		Config:         &c,
		Logger:         zap.NewNop(),
		CLFLogger:      log.New(os.Stderr, "[httpd] ", 0),
		stats:          &Statistics{},
		requestTracker: NewRequestTracker(),
	}

	// Limit the number of concurrent & enqueued write requests.
	h.writeThrottler = NewThrottler(c.MaxConcurrentWriteLimit, c.MaxEnqueuedWriteLimit)
	h.writeThrottler.EnqueueTimeout = c.EnqueuedWriteTimeout

	h.AddRoutes([]Route{
    ...
    //添加各个不同url的路由信息
    }
    
    h.AddRoutes(fluxRoute)
```
3.4 关键函数之`s.serverTCP()`，使用之前初始化的listener和handler启动真正的http服务
```
    err := http.Serve(listener, s.Handler)
	if err != nil && !strings.Contains(err.Error(), "closed") {
		s.err <- fmt.Errorf("listener failed: addr=%s, err=%s", s.Addr(), err)
	}
```

#### 连接数限制
1. 使用 `LimitListener`实现，在原始的`Listener`外包了一层还实现这个限制功能
2. `LimitListener`定义: 从下面的代码可以看出创建了一个带缓冲区的chan, 其缓冲区大小为要限制的连接数的大小
```
type limitListener struct {
	net.Listener
	sem chan struct{}
}
func LimitListener(l net.Listener, n int) net.Listener {
	return &limitListener{Listener: l, sem: make(chan struct{}, n)}
}
```
3. 接收连接：
```
func (l *limitListener) Accept() (net.Conn, error) {
	for {
		c, err := l.Listener.Accept()
		if err != nil {
			return nil, err
		}
        
        // 如果接收的连接数达到sem chan缓冲区的大小，下面这个select将进入default分支，立即close掉当前连接
        // 否则返回封装后的limitListenerConn, 它在close时调用l.release, 读取sem chan中数据，释放缓冲区空间
		select {
		case l.sem <- struct{}{}:
			return &limitListenerConn{Conn: c, release: l.release}, nil
		default:
			c.Close()
		}
	}
}
```

#### Query请求的处理流程
1. 主要实现在 `func (h *Handler) serveQuery(w http.ResponseWriter, r *http.Request, user meta.User)`

2. **调整 ResponseWriter:**  根据请求中的`Accept`头，来使用不同的ResponseWriter, 作用是设置Http Reponse中对应的`Content-Type`和格式化`Body`部分,目前支持三种类型：`text/csv`，`application/json`，`application/x-msgpack`， 具体实现可在 `services/httpd/response_writer.go`中

3. **解析http request：** 包括 uri和body部分, 最后生成 `influxql.Query`和`ExecutionOptions`
3.1 生成 influxql.Query： 通常在request uri中的`q=`是query语句，比如：select \* from m1, 会经过`influxql.NewParser`和`p.ParseQuery()`的处理
3.2 生成ExecutionOptions:
```
opts := query.ExecutionOptions{
Database:        db,
RetentionPolicy: r.FormValue("rp"),
ChunkSize:       chunkSize,
ReadOnly:        r.Method == "GET",
NodeID:          nodeID,
}
```
4. **设置closing chan**, 当当前的http连接断开时，close掉这个closing chan, 即通过当前正在处理的query请求，作相应的处理
```
var closing chan struct{}
	if !async {
		closing = make(chan struct{})
		if notifier, ok := w.(http.CloseNotifier); ok {
			// CloseNotify() is not guaranteed to send a notification when the query
			// is closed. Use this channel to signal that the query is finished to
			// prevent lingering goroutines that may be stuck.
			done := make(chan struct{})
			defer close(done)

			notify := notifier.CloseNotify()
			go func() {
				// Wait for either the request to finish
				// or for the client to disconnect
				select {
				case <-done:
				case <-notify:
					close(closing)
				}
			}()
			opts.AbortCh = done
		} else {
			defer close(closing)
		}
	}
```
5. **执行具体的query操作**: `results := h.QueryExecutor.ExecuteQuery(q, opts, closing)`, 返回`results`是个chan, 所有的query结果都从这个chan循环读取出来;
6. **非chunked方式的Response的合成**：所有结果合部缓存在内存中，从上面5中的chan循环读取出来result, 先作`h.Config.MaxRowLimit`返回行数的限制检查，再作merge,为了相同Series的数据连续存放和节省内存占用.
```
        l := len(resp.Results)
		if l == 0 {
			resp.Results = append(resp.Results, r)
		} else if resp.Results[l-1].StatementID == r.StatementID { //相同StatemnetID的result是连续返回的，中间没有间隔
			if r.Err != nil {
				resp.Results[l-1] = r
				continue
			}

			cr := resp.Results[l-1]
			rowsMerged := 0
			if len(cr.Series) > 0 {
				lastSeries := cr.Series[len(cr.Series)-1]

				for _, row := range r.Series {
					if !lastSeries.SameSeries(row) { //相同Series的row是连续返回的，中间没有间隔
						// Next row is for a different series than last.
						break
					}
					// Values are for the same series, so append them.
					lastSeries.Values = append(lastSeries.Values, row.Values...)
					rowsMerged++
				}
			}

			// Append remaining rows as new rows.
			r.Series = r.Series[rowsMerged:]
			cr.Series = append(cr.Series, r.Series...)
			cr.Messages = append(cr.Messages, r.Messages...)
			cr.Partial = r.Partial
		} else {
			resp.Results = append(resp.Results, r)
		}
```
7. **chunked方式的Response**: 从上面5中的chan循环读取出来result, 每条result立即返回到client:
```
// Write out result immediately if chunked.
		if chunked {
			n, _ := rw.WriteResponse(Response{
				Results: []*query.Result{r},
			})
			atomic.AddInt64(&h.stats.QueryRequestBytesTransmitted, int64(n))
			w.(http.Flusher).Flush()
			continue
		}
```
8. **async请求处理：** 简单讲就是不返回任何的查询结果，也就是不支持,返回的http code是`StatusNoContent`
```
if async {
		go h.async(q, results)
		h.writeHeader(w, http.StatusNoContent)
		return
	}
```

#### Write请求的处理流程
1. 写入的line protocol例子：`insert test_mea_1,tag1=v1,tag2=v2 cpu=1,memory=10`，对应到http request:
1.1 **uri部分:** `/write?consistency=all&db=my_test_db_2&precision=ns&rp=`
1.2 **body部分:** ` test_mea_1,tag1=v1,tag2=v2 cpu=1,memory=10\n`
2. 实现在 `func (h *Handler) serveWrite(w http.ResponseWriter, r *http.Request, user meta.User)`中;
2.1 解析uri和body部分:

```
    database := r.URL.Query().Get("db")
    ...
    if h.Config.MaxBodySize > 0 { //限制body读取的大小
		body = truncateReader(body, int64(h.Config.MaxBodySize))
	}
    if r.Header.Get("Content-Encoding") == "gzip" {
       //body解压缩
    }
    ...
    _, err := buf.ReadFrom(body) //读取body部分
    ...
    //解析 point
    points, parseError := models.ParsePointsWithPrecision(buf.Bytes(), time.Now().UTC(), r.URL.Query().Get("precision"))
    
    //决定多复本情况下的写入一致性策略
    level := r.URL.Query().Get("consistency")
    ...
    // 写入point
    h.PointsWriter.WritePoints(database, r.URL.Query().Get("rp"), consistency, user, points); influxdb.IsClientError(err)

    // 失败的话返回client返回信息
    h.httpError(..)
    
    // 成功时返回
    h.writeHeader(w, http.StatusNoContent)
```
#### 其他Http request请求的处理不一一详述
#### 补充一下Influxdb中的`Handler.AddRoute`的实现
1. 其作用就是添加http uri的路由信息，将相应的uri与具体的handler函数对应起来;
2. `Route`的定义
```
 type Route struct {
	Name           string
	Method         string
	Pattern        string
	Gzipped        bool
	LoggingEnabled bool
	HandlerFunc    interface{}
}

  //query请求对应的Route
   Route{
			"query", // Query serving route.
			"POST", "/query", true, true, h.serveQuery,
		}
		
	//写请求对应的Route
	Route{
		"write", // Data-ingest route.
		"POST", "/write", true, writeLogEnabled, h.serveWrite,
	}
```
3. Influxdb使用了golang提供的`net/http`包来实现它的http服务，具体的http请求都会对应到相应的`http.Handler
`, 而`http.Handler`又使用了`http.HandlerFunc`来产生，参见：[HandlerFunc](https://golang.org/pkg/net/http/#HandlerFunc), 这个`AddRout`就利用了`HandlerFunc`将handler层层包装，添加各种功能;
4. 我们来剖析一下`AddRoute`的处理流程
4.1 处理框架
```
// 针于每个route分别处理
for _, r := range routes {
        //利用route的定义和当前influxdb的config来包装生成handler
		var handler http.Handler
		... //对handler进行层层包装
		//将route和handler添加到mux, 这里这个使用了第三方的模式复用器： https://github.com/bmizerany/pat
		h.mux.Add(r.Method, r.Pattern, handler)
}
```
4.2 添加验证处理`handler = authenticate(hf, h, h.Config.AuthEnabled)
```
return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// influxdb的config里没有启动验证，走下面的逻辑
		if !requireAuthentication {
			inner(w, r, nil)
			return
		}
		
		// 验证通过会生成这个 meta.User，传过最终的请求处理函数，作授权验证
		var user meta.User

		// TODO corylanou: never allow this in the future without users
		if requireAuthentication && h.MetaClient.AdminUserExists() {
			creds, err := parseCredentials(r)
			if err != nil {
				atomic.AddInt64(&h.stats.AuthenticationFailures, 1)
				h.httpError(w, err.Error(), http.StatusUnauthorized)
				return
			}

            // http 验证支持两种，User和jwt Bearer验证，这都有对应的rfc,具体内容不展开了
			// 其中user验证又包括 basic auth和uri中自带username和password两种方式
			// 如果验证不通过，就直接返回给客户端 h.httpError(w, "xxxx", http.StatusUnauthorized)
			switch creds.Method {
			case UserAuthentication:
				...
			case BearerAuthentication:
				...
			default:
				h.httpError(w, "unsupported authentication", http.StatusUnauthorized)
			}

		}
		
		// 调用最终的请求处理函数
		inner(w, r, user)
	})
```
4.3 `handler = cors(handler)` ： 给response添加cors headers
4.4 `handler = requestID(handler)` : 给response添加request id
4.5 `handler = h.recovery(handler, r.Name)` : 在处理请求过程中捕获panic