###### Kafka Manager 简介
* [Kafka Manager](https://github.com/DavidLiuXh/kafka-manager) 可能是现在能找到的最好的可视化的Kafka管理工具, 感谢[Yahoo](https://www.yahoo.com/)-我人生中打开的一个网站-的开源;
* 使用Kafka Manager, 基本上之前需要运行Kafka相应命令行工具的工作现在都可以可视化的完成:
  1. 创建Topic, 调整消息保存时长, Partition数量等等配置;
  2. 管理Topic, 包括Reassign Partitions, Preferred Replica Election等等;
  3. 消费情况查看, 支持offset保存到zk和broker两种方式, 列出所有消费的group, 消费每个partition的详情;
 4. 集群的简单健康状态查看,包括partition分布是否均衡, leader分布是否均衡等;
 5. 通过JMX查看各种指标, 比如各个broker的网络流量和消息进出数据, 每个Topic消息的读写速度等;
* 下面我们会先简单介绍下Kafka Manager的实现和在使用中遇到的几种坑;

###### Kafka Manager实现
* 实现语言: Scala
* 用到的框架和第三方库:
 1. [Play framework](https://www.playframework.com/): Kafka-Mananger本质上是个Web应用, 因此使用play framework的MVC架构实现;
 2. [AKKA](http://akka.io): 用于构建高并发、分布式和容错的应用. Kafka Manager中的所有请求都使用akka来异步处理;
 3. [Apache Curator Framework](http://curator.apache.org/): 用于访问zookeeper;
 4. [Kafka Sdk](http://kafka.apache.org): 用于获取各Topic的last offset, 使用Admin接口实现各种管理功能; 
* 编译: 
整个工程使用 [sbt](http://www.scala-sbt.org/) 构建, 具体编译流程可以在githut上找到. sbt在build过程中会加载很多第三方依赖, 这个在国内有时会很慢, 各种同学各显神通吧.
* 实现:
  其实kafka manager的代码还是很清晰易阅读的, 如果熟悉scala和play的话应该没有难度. 不同本人也是现学现用, 好惭愧~~~. 咱们这里捡重点的说吧, 不分析具体代码实现,只讲下实现的方法:
 1. **获取集群中所有Topic**
  使用Curator访问zk获取,并监听zk相关节点 /brokers/topics 的变化;
 2. **获取Topic的partiton, leader, replicas信息**
 也是从zk获取, /brokers/topics/[topic]/partitions;
 3. **获取Topic的各partition的last offset**
 使用kafka sdk发送OffsetRequest到kafka集群来获得, 这个获取的动作会被封装成[Future[PartitionOffsetsCapture]](http://doc.akka.io/docs/akka/snapshot/scala/futures.html), 每个topic一个Future, 使用Google的[LoadingCache](https://google.github.io/guava/releases/19.0/api/docs/com/google/common/cache/LoadingCache.html)来存储这些future, 利用LoadingCache的超时淘汰机制来周期性的创建新的Future来间隔地发送OffsetRequest获取当前最新的last offset;
 4. **获取Kafka本身管理的group的消费情况**
  使用kafka sdk不断地消费"__consumer_offsets"这个topic, 来获取所有group的消费情况,关于__consumer_offsets参考 [Committing and fetching consumer offsets in Kafka](https://cwiki.apache.org/confluence/display/KAFKA/Committing+and+fetching+consumer+offsets+in+Kafka)
 5. **获取zookeeper管理的group的消费情况**
   肯定是从zk上读取, /consumers

  上面的这些实现都在 **KafkaStateActor.scala** 这个文件里.
* 各种Acotr的关系简图,仅供参考

![kafka-manager.png](http://upload-images.jianshu.io/upload_images/2020390-a6eb23318a960a68.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### Kafka Manager遇到的坑
* 多个kafka manager来管理同一个kafka集群:
   你会发现在kafka manager里无法看到所有offset使用kafka本身管理的group.
   前面我们讲过使用kafka sdk不断地消费"__consumer_offsets", 看看这段代码(在KafkaStateActor.scala中):
```
    props.put("group.id", "KafkaManagerOffsetCache")
    props.put("bootstrap.servers", bootstrapBrokerList.list.map(bi => s"${bi.host}:${bi.port}").mkString(","))
    props.put("exclude.internal.topics", "false")
    props.put("enable.auto.commit", "false")
    props.put("key.deserializer", "org.apache.kafka.common.serialization.ByteArrayDeserializer")
    props.put("value.deserializer", "org.apache.kafka.common.serialization.ByteArrayDeserializer")
    props.put("auto.offset.reset", "latest")
```

  `props.put("group.id", "KafkaManagerOffsetCache")`这句说明不管启动了几个kafka manager, 消费"__consumer_offsets"都使用同一个group.
**解决方案**: group.id从配置文件中读取,每个kafka manager使用不同的group id;

* 客户端使用某些sdk(比如librdkafka)消费topic, 客户端crash后, 在kafka manager上查看其group的消费情况, 仍然一直能看到"Consumer Instance Owner"
原因在于处理从broker返回的GroupMetadata response时没有处理异常情况:
```
              case GroupMetadataKey(version, key) =>
                    val value: GroupMetadata = readGroupMessageValue(key, ByteBuffer.wrap(record.value()))
                    value.allMemberMetadata.foreach {
                      mm =>
                        mm.assignment.foreach {
                          case (topic, part) =>
                            groupTopicPartitionMemberMap += (key, topic, part) -> mm
                        }
                    }
                }
```
这里的record.value可能为空, 此时应作清理工作:
```
                  if (null != record &&                                                                                                   
                      null != record.value()) {                                                                                           
                        val value: GroupMetadata = readGroupMessageValue(key, ByteBuffer.wrap(record.value()))                            
                        value.allMemberMetadata.foreach {                                                                                               
                          mm =>                                                                                                                         
                            mm.assignment.foreach {                                                                                                     
                              case (topic, part) =>                                                                                                     
                                groupTopicPartitionMemberMap += (key, topic, part) -> mm                                                                
                            }
                        }                                                                                                                               
                        } else {                                                                                                                          
                          groupTopicPartitionMemberMap.foreach {                                                                                          
                            case ((group, topic, part), mmd) =>                                                                                           
                              if (group == key) {                                                                                                         
                                var tmp = mmd                                                                                                             
                                tmp.memberId = ""                                                                                                         
                                tmp.clientHost = ""                                                                                                       
                                groupTopicPartitionMemberMap += (key, topic, part) -> tmp                                                                 
                              }                                                                                                                           
                          }                                                                                                                               
                        }          
```


*  Yikes! Ask timed out on [ActorSelection[Anchor(akka://kafka-manager-system/), Path(/user/kafka-manager)]] after [5000 ms] 
访问kafka manager时出现上面的超时提示, 遇到这个问题,好学不服输的你肯定会上网各种搜, 然后你会去改kafka manager的各种配置, 调大各种thread pool的容量, 增大queue size, 甚至开大jvm的使用内存, 然而问题并没有解决, 看来只剩下定时重启这一招儿了.

 **这里提供一种解决方案**: 这个超时是Actor在执行异步请求时一直等不到返回结果造成的, 主要是前面讲过的"获取Topic的各partition的last offset的Future"没有返回结果,这些Future是通过Await.ready来阻塞拿到result的, 然而在kafka manager中这个Await.ready没有给timeout, 是一直等待, 那咱们就给个timeout好了, 代码在ActorModel.scala中, 有好几处Await.ready的调用.

**找到根源:** 再也不用定时重启, 提了一个pull request到官方:[Use a separate thread to get the topic offsets to fixed bug 'Yikes! Ask timed out...'](https://github.com/yahoo/kafka-manager/pull/456), 主要就是不再使用 `Future[PartitionOffsetCapture] `来获取topic offset, 因为这个会产生大量的`Future`, 进而会产生大量的task提交到ThreadExcutor, 其实只需要启动一个单独的线程来作这件事就好了.

* Consumer offset的详情不完整
通过上面的源码分析我们知道km是通过消费"__consumer_offsets"来获取某一个组的消费情况的,消费这个topic,和消费用户自己的topic没什么两样, km里使用"props.put("auto.offset.reset", "latest")"默认offset无效时从最新位置来拉取, 如果一个group用户已经有段时间没有提交offset(但还没有完全过期), 则此时在km上看不到相应的gorup信息, 可以简单改为"props.put("auto.offset.reset", "earliest")"

* 同名group消费不同topic后,其中一个group的消费进程结束后, 仍可以看到其消费详情
 这个问题是最近被发现,之前应该是一直存在着,没能引起重视.
 这里提供一种简单的,hack的解决方案:


 ```
case GroupMetadataKey(version, key) =>
                    if (null != record &&                                                                                                   
                      null != record.value()) {                                                                                           
                        val value: GroupMetadata = readGroupMessageValue(key, ByteBuffer.wrap(record.value()))                            
                        var topicSet:Set[String] = Set()

                        value.allMemberMetadata.foreach {                                                                                               
                          mm =>                                                                                                                         
                            mm.assignment.foreach {                                                                                                     
                              case (topic, part) =>                                                                                                     
                                topicSet += topic
                                groupTopicPartitionMemberMap += (key, topic, part) -> mm                                                                
                            }
                          }

                          groupTopicPartitionMemberMap.foreach {                                                                                          
                            case ((group, topic, part), mmd) =>                                                                                           
                              if (group == key &&
                                !topicSet.contains(topic)) {                                                                                                         
                                var tmp = mmd                                                                                                             
                                tmp.memberId = ""                                                                                                         
                                tmp.clientHost = ""                                                                                                       
                                groupTopicPartitionMemberMap += (key, topic, part) -> tmp                                                                 
                              }                                                                                                                           
                          }                                                                                                                               
                        
                        } else {                                                                                                                          
                          groupTopicPartitionMemberMap.foreach {                                                                                          
                            case ((group, topic, part), mmd) =>                                                                                           
                              if (group == key) {                                                                                                         
                                var tmp = mmd                                                                                                             
                                tmp.memberId = ""                                                                                                         
                                tmp.clientHost = ""                                                                                                       
                                groupTopicPartitionMemberMap += (key, topic, part) -> tmp                                                                 
                              }                                                                                                                           
                          }                                                                                                                               
                        }                                                                                                                                 
                }

```
###### 今天就写这么多, 其他坑以后遇到再补充.

## 之前一直在写kafka的源码解析,大家有兴趣也可以指正一下 [源码解析](http://www.jianshu.com/p/aa274f8fe00f)