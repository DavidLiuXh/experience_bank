## 多数据中心部署
#### Confuent Replicator
###### 配置Repicator	
Confluent Replicator是一个Kafka connector,它运行在Kafka Connect框架内。Replicator继承了所有Kafka Connect API的优点为，包括伸缩性，性能和容错。Confluent Replicator从原始集群消费消息然后将消息写入到目标集群。这个Kafka Connect workers部署在和目标集群相同的数据中心。

下图显示了在主-从设计中这个Replicator connector运行在DC-2中，将数据单向从DC-1复制到DC-2。
![112.png](https://upload-images.jianshu.io/upload_images/2020390-0ccd0f89ec86c85e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


在双主设计中，数据是被双向复制的。一个Replicator connector将数据从DC-1复制到DC-2，另一个将数据从DC-2复制到DC-1。如下图所示：
![114.png](https://upload-images.jianshu.io/upload_images/2020390-90a964cfaf9ead6c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这两个Replictor connector的配置参数是不同，下表是它们的配置：

![115.png](https://upload-images.jianshu.io/upload_images/2020390-9ed5fc72ec380790.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


值得注意的是， Confluent Replicator自己内置了一个consumer, 默认情部下，会提交自己的consumer timestamps到原始集群。它的offsets会被转换到目标集群。在主从设计中，DC-1宕机过程中在DC-2中产生的新消息，在DC-1又恢复后需要被复制回DC-1, 这个Replicator就被用来恢复服务。然而，在双主设计中，它是不需要的，因为有两个独立的Replicator运行在双向上。为了简化双主设计的工作，我们会通过配置` offset.timestamps.commit=false`来禁止Replicator提交自己的consumer timestamps。

在双主设计中，这里有一个最小化的Replicator配置，用于从DC-1到DC-2的复制。我们假设有一个topic白名单，里面有一个topic `topic1`, 它已经在两个数据中心都配置好。如果你使用Confluent Schema Registry，这个topic 过滤器还应该包括这个topic `_schemas`,但它只需要单向复制。下面的配置允许Replicator使用message header信息来避免topic的循环复制。

![116.png](https://upload-images.jianshu.io/upload_images/2020390-dc44f13c5ca6e94f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这里有一个最小化的Replicator配置，用于从DC-2到DC-1的复制, 它和上面的配置是很相似的，除了它有不同的名字和consumer group ID, 还有它交换了` src.kafka.bootstrap.servers`和` dest.kafka.bootstrap.
servers.`的值，并且它不需要复制`_schemas`。

![117.png](https://upload-images.jianshu.io/upload_images/2020390-6ea0b94dd393ce59.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


###### 运行Replicator
这一节描述了在Kafka Connect集群内部如何将Replicator作为不同的connector来运行。为了在Kafka Connect集群里运行Replicator,你首先需要初始化Kafka Connect,在生产环境为了伸缩性和容错将总是使用分布式模式。你可以使用[ Confluent Control Center](https://www.confluent.io/confluent-control-center/)来作所有Kafka connectors的集中式管理。

![118.png](https://upload-images.jianshu.io/upload_images/2020390-58c68727de9e1aed.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


然后将Replicator connector添加到Kafka Connect。这个Repicator将运行在DC-2中将数据从DC-1拷贝到DC-2，另一个Replicator运行在DC-1中将数据从DC-2拷贝到DC-1。

![119.png](https://upload-images.jianshu.io/upload_images/2020390-a183b6539d319d35.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


你可以通过REST API和Kafka Connect交互，来管理和检查这些connector:

![120.png](https://upload-images.jianshu.io/upload_images/2020390-bbabfa5331dbe8cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


一旦运行起来，Replicator将复制过滤器允许的每一个topic:
* Topic层过滤器：在Replicator配置文件里定义的白名单，黑名单和正则。
* 消息层过滤器：使用message headers中的来源信息来自动避免两个数据中心间的循环复制。

Replicator可以很容易地运行起来，但需要确保按正确的安装和配置来部署，并且根据自己的实际环境作优化。Replicator同样支持两个数据中心间通过SSL的安全通讯。

#### Java消费应用程序
在灾难发生后，你可能需要将客户端应用程序迁移到另一个数据中心。Java消费程序在理想情况下将从之前中断的消费位置开始继续消费。如何接近这个最后消费的消息的位置，这取决于在`计算的offset的准确度`一节中描述的几个因素。

Confluent Replicator可以帮助这些consumers自动恢复到从正确的offset处开始消费，不需要开发者手动设置，需满足下列条件：
* Replicator的版本是5.0及以下
* Consumer应用程序基于Java的
* Consumer应用程序使用了Consumer Timestamps拦截器
* Consumer应用程序在两个集群中使用相现的group ID

为了使用这个特性，需要配置你的Java consumer应用程序使用Consumer Timestamps拦截器，这个拦截器在 Confluent Replicator JAR中, 名为 `kafka-connect-replicator-<release>.jar`, 针对Replicator不需要额外的配置改变。这个拦截器保存需要的timestamp信息，它在后面被 Confluent Replicator用来转换Consumer Offfset。

![121.png](https://upload-images.jianshu.io/upload_images/2020390-cc84de402d5b2bc8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### Confluent Schema 注册表
Confluent Schema注册表提供了丰富的配置项，这份白皮书只讲解几个重要的配置。首先，为每个Schema Registry实例配置一个唯一的`host.name`。我们需要改变这个参数的默认值`localhost`。这是因为其他的`Schema Registry`必须能够解析这个hostname,你也需要能够追踪哪个实例当前是主。

生产者和消费者客户端需要使用一个一致的schema ID来源，通常使用主数据中心的一个Kafka topic来作为这个来源，其topic名字通过`Kafkastore.topic`这个参数来指定。当第一个Schema Registry实例启动时，这个topic被自动创建，它有很高的持久化设置，据有三个复本，清理策略是压缩。

通过`kafkastore.bootstrap.servers`来配置所有的Schema Registry实例都使用DC-1这个Kafka 集群。需要复制schema topic到作为备份，这个在DC-1发生灾难时，你仍然可以反序列化消息。

最后，在主数据中心中配置所有的Schema Registry实例都可以参与选举成为主，他们将允许注册新的schema，配置第三个数据中心中的所有Schema Registry实例不能参与选主，禁止通过它们来注册新的schema。

![022.png](https://upload-images.jianshu.io/upload_images/2020390-65e261e17a0702bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


你同样可以对Replicator的配置作一点改变。因为DC-1是Schema Registry的主数据中心，DC-1需要将通过`Kafkastore.topic`定义的topic复制到DC-2作为备份。通常情况下，在DC-2中的`kafkastore.topic`不会被用来读或写，因为它只是DC-1的备份。在从DC-1复制数据到DC-2的过程的Replicator json配置文件中，如果你配置了任何的topic过滤器，确保它也复制了所有通过`kafkastore.topic`定义的topic(默认是_schemas)。比如，如果你已经使用了`topic.regex`参数来过滤器，需要更改它以便包含`_schemas`: `"topic.regex" : 
```
"topic.regex" : "dc1.*|_schemas"
```

一旦你在两个数据中心运行了Schema Registry,需要检查这个Schema Registry的日志信息：

* 栓查每个本地的Schema Registry 实例是否配置了正确的可以参与选主的能力：
```
$ grep master.eligibility /usr/logs/schema-registry.log
master.eligibility = true
```
* 检查哪个Schema Registry实例是被选出来的主。比如下面显示的`dc1-schema-registry`是主。如果Schema Registry实例打开了JMX ports, 你也可以通过`MBean kafka.schema.registry:type=master-slave-role.`来检查。
```
$ grep "master election" /usr/logs/schema-registry.log
[2018-09-19 13:10:31,040] INFO Finished rebalance with master election result:
Assignment{version=1, error=0, master='sr-1-4eadaf9b-ed91-4ac7-99c3-7e49187b1da3',
masterIdentity=version=1,host=dc1-schema-registry,port=8081,scheme=http,mas
terEligibility=true} (io.confluent.kafka.schemaregistry.masterelector.kafka.
KafkaGroupMasterElector)
```

## 当灾难来袭时
#### 数据中心故障
导致数据丢失或数据中心部分或全部宕机的灾难可能是由于自然灾害，攻击，掉电，外部连接丢失，物理设施故障等。当灾难来袭时，它们能够在一段不确定的时间内削弱整个数据中心的能力。针对Kafka,在这样的灾难中将发生什么呢？

![023.png](https://upload-images.jianshu.io/upload_images/2020390-f8afa00cc7848f48.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


考虑DC-1发生灾难事件时的故障转移流程。

首先，当DC-1发生故障时，客户端应用程序连接到DC-1将超时或完全失败，它们将连接到DC-2来继续生产和消费。DC-2中的Replicator也将继续运行，但因为DC-1已经故障，它将不再能从DC-1中消费到数据。

由于存在复制落后的情况，有些已经写入到原始集的消息还没有来得及到达目标集群。考虑这样一个例子，生产者将1000条消息写入到了DC-1的topic。Replicator异步将这些消息复制到DC-2，但是当灾难事件发生时，最后有一些消息没有被复制到DC-2,比如说在灾难事件发生前，只有9998条消息被复制了，这个结果就是丢失了一些消息。

![024.png](https://upload-images.jianshu.io/upload_images/2020390-716aec633b3e6395.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


因此，[监控复制的落后情况](https://docs.confluent.io/current/multi-dc-replicator/replicator-tuning.html#monitoring-replicator-lag)是非常重要的。有些监制的落后是依赖于两个数据中心间的网络延迟。如果这个落后情况持续增长，它可能是由于原始集群中数据的生产速度大于Replicator的吞吐量，你需要解决这种性能问题。

#### 客户端应用程序的故障转移
原来连接到DC-2的客户端将继续工作，它们之前就在DC-2数据中心中的客户端生产的数据和从DC-1中复制过来的数据，只是现在没有数据再从DC-1中复制过来，但消费者依然可以继续处理。

同时，原来连接到DC-1的客户端将不能再工作。取决于你的业务需求，你可以将它们继续留在DC-1中直接数据中心恢复。

换句话说，对于不能间断的业务，你可以将客户端应用程序故障转移到依旧工作的数据中心。客户端应用程序需要重新初始化并刷新新集群的metadata。需要重新配置bootstrap servers以更连接到新的集群。不论是手动还是自动故障转移到依赖于 recovery time objective (RTO)，它是灾难发生后到故障转移完成时的一个时间点。理想情况下宕机时间越短越好。如果你希望自动完成这个流程，你需要依赖于服务发现机制并在应用程序中开发故障转移逻辑。

#### 重置消费的Offsets
正如前面讨论过的，从一个正确的位置开始消费备份集群中的topic不能完全依赖于offset,因为在两个集群中同样的offset可能对应着不同的消息。而保留在消息中的时间戳在两个集群间有着相同的含义，我们可以通过时间戳来找到重新消费的位置。这个recorvy poing objective就是在灾难发生前的最近时间点，通常来说它与灾难时间越接近丢失的数据就越少。

开发者依然需要管理客户端应用程序在何时和如何在数据中心间作迁移，对于消费者来说确定从什么位置开始消费是很容易的。如果使用了Replicator的offset转换功能，消费者应用程序就可以自动确定从什么位置开始重新消费。

有些情况下，你可能需要手动重置offset。如果在灾难事件前，DC-1的消费落后了很多，如果重置到离发生灾近的时间点，就意味着有很多消息没有被消费。为了解决这个问题，你需要监控消费者的lag情况，根据这个lag情况来确定重置的时间点。

有两种方法可以重置消费者的offsets:
* 在Java客户端应用程序中使用Kafka consumer API
* 在Java客户端应用程序外使用Kafka 命令行工具

如果你希望在消费者应用程序中手动重置这个offset,你可以使用`offsetForTimes()`和`seek()`这对API的组合调用。新版本的kafka中引入了时间戳的概念，与log文件对应的索引不光有基于offset的索引，还有基于时间戳的索引，`offsetForTimes()`可以根据也时间戳找到对应的offset,以[topic, partition]为单位。`seek()`API相当于将consumer offset的位置作移动，下次拉取消息时即从移动到的新位置开始拉取。

```
// Get set of partitions assigned to consumer
final Set<TopicPartition> partitionSet = consumer.assignment();
// Initialize HashMap used for partition/timestamp mapping
final Map<TopicPartition, Long> timestampsToSearch = new HashMap<>();
// Add each partition and timestamp to the HashMap
for (TopicPartition topicPartition : partitionSet) {
 timestampsToSearch.put(topicPartition, RESET_TIME);
}
// Get the offsets for each partition
Map<TopicPartition, OffsetAndTimestamp> result = consumer.offsetsForTimes(timestampsToSearch);
// Seek to the specified offset for each partition
for (Map.Entry<TopicPartition, OffsetAndTimestamp> entry : result.entrySet()) {
 consumer.seek(entry.getKey(), entry.getValue().offset());
}
```

对于非Java客户端，可以使用kafka提供的命令行工具来重置offsets。`Kafka-consumer-groups`这个命令行工具在kafka 0.11.0版本中新添加了`--reset-offset`参数，实际上，这个重置行为会针对`s [consumer group, topic, partition] `发送一个offset更新到`__consumer_offsets`这个topic上。在这个offset重置过程中，相应的consumer group应当处于不活动状态，即它不能被使用。下面是一个例子：
```
# Reset to a specific datetime before the disaster event
$ kafka-consumer-groups --new-consumer --bootstrap-server dc2-broker2:9092 --reset-offsets
--topic dc1-topic --group my_consumer --execute --to-datetime 2017-07-05T16:34:33.236
...
TOPIC      PARTITION      NEW-OFFSET
dc1-topic     0              8000
```

#### Schema Registration
如果你使用了Schema Registry, 现在这个主实例在DC-1中不能工作了。在DC-2中从Scheme Registry实例将继续给客户端应用程序提供已经注册过的schemas和schema IDs信息，但是不能再注册新的schemas。它们也不能变为主实例，因为它们配置了`master.eligibility=false`。

在灾难期间，我们建议不要允许生产者发送需要注册新的schema的消息，并且也不要重启DC-2中的所有实例。它是基于以下两点原因作出的：
* 在主-从架构中，如果允许新的schema写到DC-2的话，会让DC-1的恢复处理流程变得更复杂，因为需要将在故障期间写入到DC-2的新schema同步回DC-1。
* 由于存在复制的延迟，存在一个小的时间窗口，在schema被提交到DC-1的kafka topic后，还没来得及被复制到DC-2。允许DC-2的Schema Registry成为主后，可能导致 Schemas丢失并且同时也不能读取应用这部分schemas的消息。 

如果注册新的schemas是必需的，那你需要在故障转移和故障恢复过程中作一些额外的事件来重新配置DC-2中的Schema Registry。在灾难事件前，DC-2中的Schema Registry实例从DC-1中读取新的schemas。现在它们需要被重定向到DC-2集群.下表中显示了Schema Registry配置的变化。你需要重启Schema Registry来使配置的改变生效。

![025.png](https://upload-images.jianshu.io/upload_images/2020390-1dd59ff7398b0c90.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 故障恢复
#### 恢复Kafka集群
当原来故障的集群从灾难事件中恢复后，你需要恢复多数据中心的配置，在两个kafka集群间同步数据并且正确地重启客户端应用程序。这个故障恢复流量看起来似乎只需要处理新数据。但实际上Kafka的故障恢复需要进一步的考虑，因为Kafka需要复制已经消费的数据和还未消费的数据。

![026.png](https://upload-images.jianshu.io/upload_images/2020390-3318f9a641c8e6d8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当原始集群恢复后，我们首先需要确保两个数据中心的zookeeper和kafka broker是否完全正常工作了。

你需要恢复Schema Registry到原始的架构，并且它的故障恢复流程取决于你在故障转移期间作了什么。如果你依然保持DC-2中的Schema Registry实例作为从实例运行并且不允许注册新的schemas,那么对于故障恢复是不需要Schema Registry作任何改变的。然而如果作为故障转移的一部分，你改变了Schema Registry的配置允许其注册新的Schemas,在恢复时你需要将所有新的schema数据从DC-2复制回DC-1。

在双主架构中，假设Replicator配置了双向复制`_schemas`topic,，那Repicator将自动将新的schemas从DC-2复制到DC-1。由于Replicator跟踪了已经处理的消息的offset,这个复制行为也将从之前中断的位置继续进行。假设Replicator是允许保留来源信息在消息中的，它也会避免循环复制。

如果在主-从架构中，假设Replicator配置了从DC-1向DC-2复制`_schemas`topic,并且一些生产者在DC-2中注册了新的schemas，那就需要在两个数据中心间同步schemas了。

#### 数据同步
在双主架构中，Replicator已经双向复制了topics, 故障恢复后数据同步是自动进行的。当DC-1完合恢复后，Replicator将自动从DC-2复制新数据到DC-1。

因此Replicator需要跟踪当前已经处理过的消息的Offsets，复制才得以从之前中断的位置继续。作为source connector的Replicator使用`offset.storate.topic`参数定义的topic来跟踪从源始集群已经消费的消息的offsets。你可以通过这个topic来监控Replicator的进度，比如下面的命令显示了消息从DC-2同步到DC-1的进度：
```
# Verify offsets tracked for this replicator
$ kafka-console-consumer --topic connect-offsets --bootstrap-server dc1-broker1:9092
--consumer-property enable.auto.commit=false --from-beginning --property print.key=true
["replicator-dc2-to-dc1",{"topic":"topic1","partition":0}] {"offset":3}
["replicator-dc2-to-dc1",{"topic":"topic1","partition":0}] {"offset":5}
```

在主-从架构中，Replicator不会自动从DC2复制新数据到DC-1，你首先需要手动来同步DC-2的最新状态到DC-1。

#### 主-从架构下的数据同步
在主-从架构下，当主集群离线时，我们可以简单地不允许生产者发送新数据到这个备份集群。但是，如果生产者应用程序需要重定向到备份数据中心时，在主集群同次上线后，我们需要将在备份集群中产生的新数据同步回主集群。

作为DC-1恢复后重新上线的一部分，如果原始集群中Kafka topic的数据已经恢复，那么仅仅在灾难发生后新产生到DC-2中的数据是需要复制回DC-1。需要配置一个新的Replicator来将DC-2的数据复制回DC-1，并且它仅仅会复制DC-2中新产生的数据，这就需要这个新的Replicator使用一个特定的consumer group id，这个group id应该和在没有发生灾难前从DC-1复制数据到DC-2中启动的Replicator的consumer group id相同。因为Replicator在运行时不光会复制数据，还会作consumer offset 的转换，在目标集群中往`__consumer_offsets`里写入和原始集群对应的记录。

如果原始集群中kafka topics的数据无法恢复，那么你需要使用DC-2中的所有数据来恢复DC-1中的数据。在运行Replicator前，先删掉DC-1中遗留的数据。这看起来有些暴力，但可以最大限度地确保没有无序的消息。有些数据可能已经被删除，这取决于主集群不可用的时长以及主集群不可用的时长与kafka 数据的保留时长之前的差距。

下一步就是将DC-2中已有的数据复制到DC-1中来恢复DC-1。DC-2中的数据是两种不同类型的数据混合在一起的：
* 在灾难发生前，从原始的DC-1中产生的被复制到DC-2中的数据，并且在消息头中被添加了来源信息
* 在灾难发生后，新生产到DC-2中的数据，不是复制而来的并且也没有来源头信息

如果你使用默认的过滤行为来运行Repicator, 在灾难发生前产生到DC-1中的数据将不会被复制回DC-1。这是因为这部分消息的消息头中的来源信息已经表明其来自DC-1集群，为避免循环复制，它将不会再被复制。

为了改变这个默认的行为，拷贝DC-2中所有的消息到DC-1中，包括有和没有信息来源头信息的消息，可以单独运行一个Replicator实例，并且配置这个`provenance.header.filter.overrides`,它的格式是一个分号分隔的三元素的集合列表：
1. cluster id的正则表达式;
2. topic名字的正则表达式;
3. 时间戳范围

下面显示了一个配置的实例：
```
provenance.header.enable=true
provenance.header.filter.overrides=DC-1,_schemas,-;DC-1,topic1,0-1539634429528
```

实时监控这个Replicator的进度并且等待复制的lag到达0,这表时DC-1已经完全追上了DC-2中的数据。在这些消息中已经保留着所有的header信息，包括来源信息。然后停止这个临时的Replicator实例并且恢复正常的操作，完全数据的同步。

###### 消息顺序
在数据同步后，两个集群中给定topic partition中消息的顺序可能是不一致的。这是因为，在发生灾难时，由于延迟，DC-1中还有数据没有复制到DC-2中。为了说明这一点，我们来看之前的一个例子，在DC-1中有10000条消息，然后有9998条复制到了DC-2中。故障发生后，有新数据写入到了DC-2相同的topic中，然后故障恢复后，Replicator会继续复制10000条消息的最后两条到DC-2中，这导致了在两个数据中心后，消息顺序的不一致。

##### 客户端应用程序重启
一旦故障恢复，数据同步完成，客户端应用程序将要切换完原始主集群，需要重新初始化来连接到原始主集群。

## 总结
这份白皮书讨论了架构，配置等构建模块和后续的故障转移，故障恢复流程。这里涉及了多种针对多数据中心架构的用户场景，但焦点集中在灾难恢复上。你的架构将非常依赖于你的业务需求。你需要应用合适的构建模块来增加你的灾难恢复预案。