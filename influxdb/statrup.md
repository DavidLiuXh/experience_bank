
#### Influxdb启动流程
1. Influxdb的启动代码实现在 `cmd/influxd/main.go`中
2. `influxd`支持下面几种启动命令
```
 backup               downloads a snapshot of a data node and saves it to disk 
 config               display the default configuration                        
 help                 display this help message                                
 restore              uses a snapshot of a data node to rebuild a cluster      
 run                  run node with existing configuration                     
 version              displays the InfluxDB version                            
```
我们启动的话通常是 `influxd run -config [config file path]`
3. 简单看一下`run`命令相关的代码:
```
func (m *Main) Run(args ...string) error {
	name, args := cmd.ParseCommandName(args)

	// Extract name from args.
	switch name {
	case "", "run":
		cmd := run.NewCommand()

		// Tell the server the build details.
		cmd.Version = version
		cmd.Commit = commit
		cmd.Branch = branch

		if err := cmd.Run(args...); err != nil {
			return fmt.Errorf("run: %s", err)
		}

		signalCh := make(chan os.Signal, 1)
		signal.Notify(signalCh, os.Interrupt, syscall.SIGTERM)
		cmd.Logger.Info("Listening for signals")

		// Block until one of the signals above is received
		<-signalCh
		cmd.Logger.Info("Signal received, initializing clean shutdown...")
		go cmd.Close()

		// Block again until another signal is received, a shutdown timeout elapses,
		// or the Command is gracefully closed
		cmd.Logger.Info("Waiting for clean shutdown...")
		select {
		case <-signalCh:
			cmd.Logger.Info("Second signal received, initializing hard shutdown")
		case <-time.After(time.Second * 30):
			cmd.Logger.Info("Time limit reached, initializing hard shutdown")
		case <-cmd.Closed:
			cmd.Logger.Info("Server shutdown completed")
		}
        ...
	}

	return nil
}
```
主要就是`cmd := run.NewCommand()`创建cmd对象，然后调用其`Run`方法
3. 我们来看一下`Command.Run`的实现
```
func (cmd *Command) Run(args ...string) error {
    // 解析参数
    options, err := cmd.ParseFlags(args...)

    ...
	
    // 解析配置文件，初始化各组件的配置信息
	config, err := cmd.ParseConfig(options.GetConfigPath())
     
	// 初始化logger
	if cmd.Logger, logErr = config.Logging.New(cmd.Stderr); logErr != nil {
		// assign the default logger
		cmd.Logger = logger.New(cmd.Stderr)
	}
	
	// 如果配置了pid file path, 就写pud
	cmd.writePIDFile(options.PIDFile)
	
	// 创建Server对象，并调用Open方法将 Server运行起来
	s, err := NewServer(config, buildInfo)
	...
	if err := s.Open(); err != nil {
		return fmt.Errorf("open server: %s", err)
	}

    // 开如monitor server error信息
    go cmd.monitorServerErrors()
}
```
4. 我们来过一下`NewServer`的实现， 它主要的功能就是依据配置Server对象和它管理的各个组件,  主要包括
```
Monitor
MetaClient
TSDBStore
Subscriber
PoinitsWriter
QueryExecutor
...
```
5. 紧接着会调用`Server.Open`添加各种service,让各个组件运行起来
```
// Open opens the meta and data store and all services.
func (s *Server) Open() error {
	// 创建并运行一个tcp的连接复用器
	ln, err := net.Listen("tcp", s.BindAddress)
	if err != nil {
		return fmt.Errorf("listen: %s", err)
	}
	s.Listener = ln

	// Multiplex listener.
	mux := tcp.NewMux()
	go mux.Serve(ln)

	// 添加各种service
	s.appendMonitorService()  // 
	s.appendPrecreatorService(s.config.Precreator) //预创建ShardGroup
	s.appendSnapshotterService() //使用上面的tcp连接复用器，处理snapshot相关的请求
	s.appendContinuousQueryService(s.config.ContinuousQuery) // 连续query服务
	s.appendHTTPDService(s.config.HTTPD) //http服务，接收并处理所有客户端的请求
	s.appendRetentionPolicyService(s.config.Retention) //依据RetentionPolicy周期性的作清理
	
	// Graphite, Collectd, OpenTSDB都会对其实TSDB数据格式的支持
	for _, i := range s.config.GraphiteInputs {
		if err := s.appendGraphiteService(i); err != nil {
			return err
		}
	}
	for _, i := range s.config.CollectdInputs {
		s.appendCollectdService(i)
	}
	for _, i := range s.config.OpenTSDBInputs {
		if err := s.appendOpenTSDBService(i); err != nil {
			return err
		}
	}
	for _, i := range s.config.UDPInputs {
		s.appendUDPService(i)
	}

	...
	
	// Open TSDB store.
	if err := s.TSDBStore.Open(); err != nil {
		return fmt.Errorf("open tsdb store: %s", err)
	}

	// Open the subscriber service
	if err := s.Subscriber.Open(); err != nil {
		return fmt.Errorf("open subscriber: %s", err)
	}

	// Open the points writer service
	if err := s.PointsWriter.Open(); err != nil {
		return fmt.Errorf("open points writer: %s", err)
	}

	s.PointsWriter.AddWriteSubscriber(s.Subscriber.Points())

	for _, service := range s.Services {
		if err := service.Open(); err != nil {
			return fmt.Errorf("open service: %s", err)
		}
	}

	...

	return nil
}
```

#### 图解Influxd的启动流程
![influxdb_run.png](https://upload-images.jianshu.io/upload_images/2020390-f7633e0f2657e60d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
