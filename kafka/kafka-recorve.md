* Kafka 集群部署环境
  1. kafka 集群所用版本 0.9.0.1
  2. 集群部署了实时监控:  通过实时写入数据来监控集群的可用性, 延迟等;
---
## 集群故障发生
* 集群的实时监控发出一条写入数据失败的报警, 然后马上又收到了恢复的报警, 这个报警当时没有重要,没有去到对应的服务器上去看下log,  *恶梦的开始啊~~~*
*  很快多个业务反馈Topic无法写入, 运维人员介入

## 故障解决
* 运维人员首先查看kafka broker日志, 发现大量如下的日志:
```
[2017-10-12 16:52:38,141] ERROR Processor got uncaught exception. (kafka.network.Processor)
java.lang.ArrayIndexOutOfBoundsException: 18
        at org.apache.kafka.common.protocol.ApiKeys.forId(ApiKeys.java:68)
        at org.apache.kafka.common.requests.AbstractRequest.getRequest(AbstractRequest.java:39)
        at kafka.network.RequestChannel$Request.<init>(RequestChannel.scala:79)
        at kafka.network.Processor$$anonfun$run$11.apply(SocketServer.scala:426)
        at kafka.network.Processor$$anonfun$run$11.apply(SocketServer.scala:421)
        at scala.collection.Iterator$class.foreach(Iterator.scala:742)
        at scala.collection.AbstractIterator.foreach(Iterator.scala:1194)
        at scala.collection.IterableLike$class.foreach(IterableLike.scala:72)
        at scala.collection.AbstractIterable.foreach(Iterable.scala:54)
```
* 这个问题就很明了了, 在之前的文章里有过介绍: [Kafka运维填坑](https://www.jianshu.com/p/d2cbaae38014), 上面也给出了简单修复, 主要原因是 ***新版kafka 客户端 sdk访问较旧版的kafka, 发送了旧版 kafka broker 不支持的request***, 这会导致exception发生, 然后同批次select出来的所有客户端对应的request都将被抛弃不能处理,代码在 `SocketServer.scala`里面, 大家有兴趣可以自行查阅
  1. 这个问题不仅可能导致客户端的request丢失, broker和broker, broker和controller之间的通讯也受影响;
  2. 这也解释了为什么 实时监控 先报警 然后又马上恢复了: 不和这样不被支持的request同批次处理就不会出现问题;
* 解决过程:
   1. 我们之前已经修复过这个问题, 有准备好的相应的jar包;
   2. 运维小伙伴开始了愉快的jar包替换和启动broker的工作~~~~~~
## 集群恢复
* kafka broker的优雅shutdown的时间极不受控, 如果强行kill -9 在start后要作长时间的recovery, 数据多的情况下能让你等到崩溃;
* 集群重启完, 通过log观察, `ArrayIndexOutOfBoundsException`异常已经被正确处理, 也找到了相应的业务来源;
* 业务反馈Topic可以重新写入;
--- 
***然而, 事件并没有结束, 而是另一个恶梦的开始***
---
## 集群故障再次发生
* 很多业务反馈使用原有的group无法消费Topic数据;
* 用自己的consumer测试, 发现确实有些group可以, 有些group不能消费;
* *一波不平一波又起, 注定是个不平凡的夜晚啊, 居然还有点小兴奋~~~*
## 故障解决
* **查看consumer测试程序不能消费时的日志,一直在重复如下log**:
```
Group "xxx" coordinator is xxx.xxx.xxx.xxx:9092 id 3
Broker: Not coordinator for group
```
1. 第一条日志 说明consumer已经确认了当前的coordinator, 连接没有问题;
2. 第二条日志显示没有 ` Not coordinator`, 对应broker端是说虽然coordinator确认了,但是没有在这个 coodinator上找到这个group对应的metada信息;
3. group的metada信息在coordinator启动或__consuser_offsets的partion切主时被加载到内存,这么说来是相应的__consumer_offsets的partition没有被加载;
4. 关于coordinator, __consumer_offsets, group metada的信息可以参考 [Kafka的消息是如何被消费的?](https://www.jianshu.com/p/1aba6e226763)
* **查看broker端日志, 确认goroup metadata的相关问题**
1. 查找对应的__consumer_offsets的partition的加载情况, 发现对应的__consumer_offsets正在被`Loading`;
```
Loading offsets and group metadata from [__consumer_offsets,19] (kafka.coordinator.GroupMetadataManager)
```
   2. 没有找到下面类似的加载完成的日志:
```
Finished loading offsets from [__consumer_offsets,27] in 1205 milliseconds. (kafka.coordinator.GroupMetadataManager)
```
也没有发生任何的exception的日志
3. **使用jstack来dump出当前的线程堆栈多次查看, 证实一直是在加载数据,没有卡死;
* 现在的问题基本上明确了, 有些__consumer_offsets加载完成了,可以消费,  些没有完成则暂时无法消费, 如果死等loading完成, 集群的消费可以正常, 但将花费很多时间;**

* **为何loading这些__consumer_offsets要花费如此长的时间?**
  1. 去到__conuser_offsets partition相应的磁盘目录查看,发生有2000多个log文件, 每个在100M左右;
  2. kaka 的log compac功能失效了,  这个问题在之前的文章里有过介绍: [Kafka运维填坑](https://www.jianshu.com/p/d2cbaae38014),
  3. log compact相关介绍可以参考 [Kafka的日志清理-LogCleaner](https://www.jianshu.com/p/3e0d426313aa)

* **手动加速Loading**:
   1. 即使log cleaner功能失败, 为了加速loading, 我们手动删除了大部分的log文件; 这样作有一定风险, 可能会导致某些group的group metadata和committed offset丢失, 从而触发客户端在消费时offset reset;
 ## 故障恢复
* 所有__consumer_offset都加载完后, 所有group均恢复了消费;
---
## 总结
* 对实时监控的报警一定要足够重视;
* 更新完jar包, 重启broker时, 三台存储__consumer_offsets partition合部同时重启,均在Loading状态, 这种作法不合适,最多同时重启两台, 留一台可以继续提供coordinattor的功能;
* 加强对log compact失效的监控, 完美方案是找到失效的根本原因并修复;
---
# [Kafka源码分析-汇总](http://www.jianshu.com/p/aa274f8fe00f)