[TOC]

## Query请求的执行流程分析
* 我们以 `httpd/handler.go`中的`serverQuery`为入口来分析;
* 在前面我们有专门讲解 [httpd/handler]() 的一篇文章;
* 我们不会分析查询结果是如何通过tsm tree和倒排索引得到的，重点放在查询的上层流程上;
* 本章我们将主要精力放在 `query.Executor`的分析上。

#### `Executor`的定义和创建
1. 定义：
```
type Executor struct {
	// Used for executing a statement in the query.
	// 具体的查询操作
	StatementExecutor StatementExecutor

	// Used for tracking running queries.
	// 每个query都会对应一个task, 由TaskManager统一管理
	TaskManager *TaskManager

	// Logger to use for all logging.
	// Defaults to discarding all log output.
	Logger *zap.Logger

	// expvar-based stats.
	stats *Statistics
}
```
2. 对应的New函数:
```
func NewExecutor() *Executor {
	return &Executor{
		TaskManager: NewTaskManager(),
		Logger:      zap.NewNop(),
		stats:       &Statistics{},
	}
}
```
3. 创建和初始化在`run/server.go`中的`NewServer`函数， 其中包括`TaskManager`和`StatementExecutor`的初始化
```
	s.QueryExecutor = query.NewExecutor()
	
	s.QueryExecutor.StatementExecutor = &coordinator.StatementExecutor{
		MetaClient:  s.MetaClient,
		TaskManager: s.QueryExecutor.TaskManager,
		TSDBStore:   s.TSDBStore,
		ShardMapper: &coordinator.LocalShardMapper{
			MetaClient: s.MetaClient,
			TSDBStore:  coordinator.LocalTSDBStore{Store: s.TSDBStore},
		},
		Monitor:           s.Monitor,
		PointsWriter:      s.PointsWriter,
		MaxSelectPointN:   c.Coordinator.MaxSelectPointN,
		MaxSelectSeriesN:  c.Coordinator.MaxSelectSeriesN,
		MaxSelectBucketsN: c.Coordinator.MaxSelectBucketsN,
	}
	s.QueryExecutor.TaskManager.QueryTimeout = time.Duration(c.Coordinator.QueryTimeout)
	s.QueryExecutor.TaskManager.LogQueriesAfter = time.Duration(c.Coordinator.LogQueriesAfter)
	s.QueryExecutor.TaskManager.MaxConcurrentQueries = c.Coordinator.MaxConcurrentQueries
```

#### `TaskManager`相关内容解析
##### `Task`分析
1. 首先我们先来看`Task`, 它被定义在`queyr/executor.go`, 每个Query请求都会对应一个`Task`,交由`TaskManager`统一管理
```
type Task struct {
	query     string  //Query请求的string
	database  string  //当前Query需要操作的db
	status    TaskStatus //task运行的状态： RunningTask或KilledTask
	startTime time.Time // task开始时间
	closing   chan struct{} // task结束时，通过这个closing chan来通知
	monitorCh chan error
	err       error
	mu        sync.Mutex
}
```
2. `monitor`监控函数，用来也监控task来发生的事情，比如慢请求
```
func (q *Task) monitor(fn MonitorFunc) {
	if err := fn(q.closing); err != nil {
		select {
		case <-q.closing:
		case q.monitorCh <- err:
		}
	}
}
```
2.1 这个`MonitorFunc`是一个函数类型，定义为`type MonitorFunc func(<-chan struct{}) error`, 它用来检查当前task对应的query的健康情况，如果当前query被某些错误中断，它将返回err;
2.2 如果`fn MonitorFunc`返回了err, 则将此err写到q.monitorCh这个chan中;
3. `close`函数，结束掉一个task
```
	q.mu.Lock()
	if q.status != KilledTask {
		// Set the status to killed to prevent closing the channel twice.
		q.status = KilledTask
		//通过q.closing这个chan作通知
		close(q.closing)
	}
	q.mu.Unlock()
```
3. `kill`函数，task自杀
```
	q.mu.Lock()
	if q.status == KilledTask {
		q.mu.Unlock()
		return ErrAlreadyKilled
	}
	q.status = KilledTask
	close(q.closing)
	q.mu.Unlock()
	return nil
```
##### `ExecutionContext`分析
1. 定义在`execution_context.go`中, 跟踪当前query的执行状态
```
type ExecutionContext struct {
    // 匿名字段 Context 
	context.Context

	// The statement ID of the executing query.
	statementID int

	// The query ID of the executing query.
	QueryID uint64

	// The query task information available to the StatementExecutor.
	task *Task

	// Output channel where results and errors should be sent.
	Results chan *Result

	// Options used to start this query.
	//在 httpd.Handler.go中生成的ExecutionOptions
	ExecutionOptions

	mu   sync.RWMutex
	done chan struct{}
	err  error
}
```
2. `watch`	函数: 开一个新的goroutine, 等待task.closing chan的通知，Contex.Done完成的通知和ExecutionOptions.AbortCh的通知
```
func (ctx *ExecutionContext) watch() {
	ctx.done = make(chan struct{})
	if ctx.err != nil {
		close(ctx.done)
		return
	}

	go func() {
		defer close(ctx.done)

		var taskCtx <-chan struct{}
		if ctx.task != nil {
			taskCtx = ctx.task.closing
		}

		select {
		case <-taskCtx:
			ctx.err = ctx.task.Error()
			if ctx.err == nil {
				ctx.err = ErrQueryInterrupted
			}
		case <-ctx.AbortCh:
			ctx.err = ErrQueryAborted
		case <-ctx.Context.Done():
			ctx.err = ctx.Context.Err()
		}
	}()
}
```

##### TaskManager分析
1. 定义在`query/taks_manager.go`
```
type TaskManager struct {
	// Query 执行的超时时长，超时请求的执行将被中断
	QueryTimeout time.Duration

	// Log queries if they are slower than this time.
	// If zero, slow queries will never be logged.
	// 慢请求的阈值
	LogQueriesAfter time.Duration

	// Maximum number of concurrent queries.
	// 并发处理的query数量
	MaxConcurrentQueries int

	// Logger to use for all logging.
	// Defaults to discarding all log output.
	Logger *zap.Logger

	// Used for managing and tracking running queries.
	// Task id和task组成的map
	queries  map[uint64]*Task
	nextID   uint64
	mu       sync.RWMutex
	shutdown bool
}
```
2. `TaskManager.AttachQuery`: 将query封装成`task`交由`TaskManager`管理
```
func (t *TaskManager) AttachQuery(q *influxql.Query, opt ExecutionOptions, interrupt <-chan struct{}) (*ExecutionContext, func(), error) {
    ...

    // 超过设置的并发Query数量后，Attach失败
	if t.MaxConcurrentQueries > 0 && len(t.queries) >= t.MaxConcurrentQueries {
		return nil, nil, ErrMaxConcurrentQueriesLimitExceeded(len(t.queries), t.MaxConcurrentQueries)
	}

    // 生成 Task和 TaskId
	qid := t.nextID
	query := &Task{
		query:     q.String(),
		database:  opt.Database,
		status:    RunningTask,
		startTime: time.Now(),
		closing:   make(chan struct{}),
		monitorCh: make(chan error),
	}
	t.queries[qid] = query

    //新开一个goroutine, 等待query超时，http连接断后工等各种情况,后面详述
	go t.waitForQuery(qid, query.closing, interrupt, query.monitorCh)
	
	if t.LogQueriesAfter != 0 {
	    // 遇到慢查询打log
		go query.monitor(func(closing <-chan struct{}) error {
			timer := time.NewTimer(t.LogQueriesAfter)
			defer timer.Stop()

			select {
			case <-timer.C:
				t.Logger.Warn(fmt.Sprintf("Detected slow query: %s (qid: %d, database: %s, threshold: %s)",
					query.query, qid, query.database, t.LogQueriesAfter))
			case <-closing:
			}
			return nil
		})
	}
	t.nextID++

    // 生成ExcutionContext
	ctx := &ExecutionContext{
		Context:          context.Background(),
		QueryID:          qid,
		task:             query,
		ExecutionOptions: opt,
	}
	
	// 开如watch这个query的执行状态
	ctx.watch()
	return ctx, func() { t.DetachQuery(qid) }, nil
}
```

##### 分析Query执行过程中可能遇到的几种情况
###### 前提
* 其实还是从`results := h.QueryExecutor.ExecuteQuery(q, opts, closing)`说起

###### AttachQuery失败
1. 在执行Query前，先要将Query生成Task交由`TaskManager`管理，AttachQuery失败有两种情况
```
    // query/task_manager.go:AttachQuery
	if t.shutdown {
		return nil, nil, ErrQueryEngineShutdown
	}

	if t.MaxConcurrentQueries > 0 && len(t.queries) >= t.MaxConcurrentQueries {
		return nil, nil, ErrMaxConcurrentQueriesLimitExceeded(len(t.queries), t.MaxConcurrentQueries)
	}
```
这将反回相应的err, 如果处理这个err呢？
```
// query/executor.go:executeQuery
func (e *Executor) executeQuery(query *influxql.Query, opt ExecutionOptions, closing <-chan struct{}, results chan *Result) {
    ...
    ctx, detach, err := e.TaskManager.AttachQuery(query, opt, closing)
	if err != nil {
		select {
		case results <- &Result{Err: err}:
		case <-opt.AbortCh:
		}
		return
	}
    ...
}
```
这个err被封装到`Result`中写入到results这个chan中. 接下来呢？
```
results := h.QueryExecutor.ExecuteQuery(q, opts, closing)
for r := range results {
  ...
}
```
调用返回，results就是上面写入的chan（httpd/henalder.go:serveQuery(), 读取出包含err信息的Result返回给客户端

###### Query被正确执行并返回给客户端
1. `AttachQuery` 成功后返回了`ExecutionContext`,并且将返回Query结果的Results chan赋值给`ExcutionContext.Results`备用;
2. 一个Query可能包含多个statement, 逐一执行
```
for ; i < len(query.Statements); i++ {
...
}
```
3. Query语句过滤, 针对`system measurements（_fieldKeys，_measurements,_series,_tagkeys,_tags）`的 select操作，是不被允许的， 将含err信息的Result写入Results
```
results <- &Result{
		Err: fmt.Errorf("unable to use system source '%s': use %s instead", s.Name, command),
}
```
4. 改变query statement, 主要是针对`meta`的`show ...`，改写成针对`system measurement`的select语句
5. 执行具体的Query: `err = e.StatementExecutor.ExecuteStatement(stmt, ctx)`(coordinator/statement_executor.go),特别是针对select query, 调用 `func (e *StatementExecutor) executeSelectStatement(stmt *influxql.SelectStatement, ctx *query.ExecutionContext) error`
关于`StatementExecutor`， 我们这里不详细分析，只需要知道它会将查询结果封装成query.Result里写入到上面提到的contex.Results chan中

###### Query执行过程中Http连接中断
1. 在 `httpd/henalder.go:serveQuery中`
```
closing = make(chan struct{})
		if notifier, ok := w.(http.CloseNotifier); ok {
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
```
如果http 连接断掉，则会`close(closing)`，关掉这个chosing chan;
2. 在 `TaskManager.AttachQuery`时，会`TaskManager.waitForQuery`:
```
 select {
	case <-closing:
		t.queryError(qid, ErrQueryInterrupted)
	    ....
	}
	t.KillQuery(qid)
```
上面的select会返回，`KillQuery`被调用，它又会调用`Task.kill()`
```
 func (q *Task) kill() error {
	...
	q.status = KilledTask
	close(q.closing)
	...
}
```
将`q.closing`这个chan关掉，让我们再次回到`AttachQuery`的最后是`ExcutionContext.watch`:
```
func (ctx *ExecutionContext) watch() {
    ctx.done = make(chan struct{})
	...
	go func() {
		defer close(ctx.done)

		var taskCtx <-chan struct{}
		if ctx.task != nil {
			taskCtx = ctx.task.closing
		}

		select {
		case <-taskCtx:
			ctx.err = ctx.task.Error()
			if ctx.err == nil {
				ctx.err = ErrQueryInterrupted
			}
		...
		}
	}()
}
```
上面的select将被触发,将`ctx.err = ErrQueryInterrupted`并调用`close(ctx.done)`，关掉这个ctx.donw chan,这个chan很关键，让我们回到执行具体query的`coordinator/statement_executor.go:executeSelectStatement`:
```
 func (e *StatementExecutor) executeSelectStatement(stmt *influxql.SelectStatement, ctx *query.ExecutionContext) error {
    ...
	for {
	select {
	            ...
				case <-ctx.Done():
					return ctx.Err()
				default:
				}
				...
	}
	...
	if err := ctx.Send(result); err != nil {
			return err
		}
}
```
上面的两处都会返回err, `executeSelectStatement`调用结束，返回err -> ErrQueryInterrupted, 最终被封装在`query.Result`里写入到`Results chan`中;

###### Query执行超时
1. 如果超时，`TaskManager::waitForQuery`中下面的代码将被触发:
```
var timerCh <-chan time.Time
	if t.QueryTimeout != 0 {
		timer := time.NewTimer(t.QueryTimeout)
		timerCh = timer.C
		defer timer.Stop()
	}

	select {
	...
	case <-timerCh:
		t.queryError(qid, ErrQueryTimeoutLimitExceeded)
	...
	}
	t.KillQuery(qid)
```
2. 往下的流程就和http断开后的流程一样了，最后返回的err -> ErrQueryTimeoutLimitExceeded