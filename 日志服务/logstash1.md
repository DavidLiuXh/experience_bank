* 基于Logstash 5.4.0版本
* 主要针对收集本地文件日志后写入kafka这个场景
* 还在进一步使用中, 遇到的新的问题会持续补充
---

######  无法写入kafka集群
- **现象:** 可以从本地要收集的文件中读取文件内容,但无法写入kafka集群;
- **原因:** kafka 集群版本为0.9.0.1,  Logstash中自带的kafka client jar包不兼容, [官方文档](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-kafka.html)其实有说明
- **解决方案:** 使用kafka 0.9.0.1版本中的kafka client jar作替换,主要涉及到下面的两个jar包, 替换后名字还要保持 **kafka-clients-0.10.0.1.jar**
 ``` 
/vendor/bundle/jruby/1.9/gems/logstash-input-kafka-5.1.6/vendor/jar-dependencies/runtime-jars/kafka-clients-0.10.0.1.jar
./vendor/bundle/jruby/1.9/gems/logstash-output-kafka-5.1.5/vendor/jar-dependencies /runtime-jars/kafka-clients-0.10.0.1.jar
```
###### 同时收集多个文件时,有些文件收集很慢或无法收集
- **现象:** file input的path匹配到了多个待收集的文件, 但有些文件收集很慢或无法收集
- **原因:** 简单讲file input plugin使用filewatch组件来轮询文件的变化进行文件收集, filewatch发现文件有新数据可收集时会使用`loop do end`循环来一直读取当前文件, 直到收集到文件尾或有异常发生,才退出;
如此这样, 当有一个很大的或频繁被写入文件先处于被收集状态, 则其他待收集文件则没有机会被收集;
当然作者设计这样的逻辑也有他的道理.
- **解决方案:** 解决起来也很简单, 既然是轮询文件的变化进行文件收集, 这个`loop do end`循环是在`observe_read_file`这个函数里(./vendor/bundle/jruby/1.9/gems/filewatch-0.9.0/lib/filewatch/observing_tail.rb), 可以增加一个行数控制, 每次当当前文件收集的行数大于预设的阈值后就跳出这个`loop do end`循环.
###### 无法退出Logstash进程之一
- **现象:** `kill -SIGTERM`后,logstash进程一直无法结束, 日志里会报`The shutdown process appears to be stalled due to busy or blocked plugins. Check the logs for more information`
- **原因:** file plugin中有线程没有结束, 经排查后发现和上一个问题的原因是一样的, 正在收集一个最大的文件, 陷入了`loop do end`循环.
- **解决方案:** 引入一个变量, 进程退出时此变量被set, 然后在 `loop do end`循环中check这个变量, 来决定是否退出这个循环.
###### 无法退出Logstash进程之二
- **现象:** `kill -SIGTERM`后,logstash进程一直无法结束, 日志里会报`The shutdown process appears to be stalled due to busy or blocked plugins. Check the logs for more information`
- **原因:** 其实还是有线程没有结束掉所致, 经排查问题出在`/vendor/bundle/jruby/1.9/gems/logstash-codec-multiline-3.0.3/lib/logstash/codecs/identity_map_codec.rb`这个文件中的`start`和`stop`函数, 按现在的逻辑`stop`后`start`仍可能被调用, 然后在`start`里又开启了一个新的thread, 却没有机会被stop了;
- **解决方案:** 引入一个变量, 确何在`stop`后, 即使再次调用`start`, 也不会再开启一个新的线程.
###### 运行一段时间后,发现向kafka发送数据特别慢, 文件数据积压很多
* **现象**:运行一段时间后,发现向kafka发送数据特别慢, 文件数据积压很多, 重启后又恢复正常;
* **原因**: 在发现此问题是jstack打印出logstash当前所有线程的堆栈, 发现kafka发送的相关线程都卡在kafka java sdk里的`BufferPoll::allocate`, 具体原因可参考[kafka官方bug](https://issues.apache.org/jira/browse/KAFKA-3651) 
- **解决方案:** 因为我们的kafka版本是0.9.0.1, logstash中我们也是用了对应的sdk版本, 手动merge了官方的修复,替换kafka sdk jar, 测试目前没有问题
---
# [Logstash源码分析-框架概述](http://www.jianshu.com/p/8e511e61d0f0)