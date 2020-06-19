[toc]

>译者注：
>这篇文章翻译自 https://www.confluent.io/blog/disaster-recovery-multi-datacenter-apache-kafka-deployments 中提供的白皮书。尽管它其中推荐使用的都是Confluent Platform提供的工具，但其对相关问题处理的思路和想法我们完全可以借用，还是据有一定的参考价值。


# 使用多数据中心部署来应对灾难恢复

## 简介
数据中心宕机和数据丢失能导致企业损失很多收入或者完全停摆。为了将由于事故导致的宕机和数据丢失带来的损失最小化，企业需要制定业务可持续性计划和灾难恢复策略。

灾难恢复计划经常需要在多个数据中心部署Apache Kafka, 且这些数据中心在地理位置上是分散的。如果灾难来袭，比如说致命的硬件故障，软件故障，电源掉电，拒绝式服务攻击和其他任何可能的事件，导致一个中心数据完成无法工作，Kafka应该继续不间断地运行在另一个数据中心直至服务恢复。

这份白皮书提供了一套基于Confluent Platform平台能力和Apache Kafka主要发行版本所作出的灾难恢复方案的概要。Confluent Platform 提供了下列构建模块：
* 多数据中心设计
* 中心化的schema管理
* 避免消息被循环复制的策略
* 自动转换consumer offset

这份白皮书将使用上述构建模块来介绍如何配置和启动基于多数据中心的Kafka集群部署，并且告诉你如果一个中心数据不可用将要作什么，如果这个中心数据又恢复了将如何作复原操作。

你可能正在考虑主-从方案（数据在kafka集群间单向复制），双主方案（数据在kafka集群间双向复制），客户端可以仅从本地集群也可以从本地和远端两个集群读取数据，服务发现机制允许作自动故障转移和基于不同地理位置提供服务等。你的架构将非常依赖于你的商业需求，但是你可以使用这份白皮书里的构建模块来增强你的灾难恢复计划。

## 架构
#### 单一数据中心
首先，让我们一起看下在单数据中心部署的Kafka集群是如何提供消息的持久化的。下面的图显示了单数据中心的架构：
<div align=center>![kafka-single-datacenter](file:///home/lw/图片/kafka-single.png)</div>

在单数据中心情况下，Kafka集群内部的数据复制是实现消息持久化的基本方法。生产者写入数据到集群，然后消费者从partition的leader读取数据。数据从主节点同步复制到从节点以确保消息在不同的broker上有多份拷贝。Kafka生产者能够通过设置`Ack`这个写入配置参数来控制写入一致性的级别。

生产者设置`Ack=All`, 将为数据的复制提供了最强有效的保证，它确保在leader broker给生产者发送response前，集群里其他的作为复本的broker都Ack了接收到的数据。如果leader broker故障，其余的follower broker将重新选举出主，此时这个Kafka集群将恢复并且客户端将能够通过新的leader继续读取消息。在 Kafka 0.11版本中，引入了 [KIP-101](https://cwiki.apache.org/confluence/display/KAFKA/KIP-101+-+Alter+Replication+Protocol+to+use+Leader+Epoch+rather+than+High+Watermark+for+Truncation) 这个改进，它加强了集群内部的数据复制协议，解决了之前存在的一些问题，提供了容错能力。

另外，客户端可以通过任何的broker集合连接到Kafka集群，这个用于连接的broker集群叫作`bootstrap brokers`，因为集群内任一台broker上都缓存了整个集群完整的meta data信息，所以客户端连接到任何一台，都可以收取到所有的meta data。如果客户端使用某一台具体的broker连接到集群，但这台broker正好发生故障，那客户端依然可以使用这组`bootstrap brokers`中的其他broker连接到该集群。

对于Zookeeper, 我们建议至少部署3个节点来维护在有节点发生故障时的高可用性。

最后，我们还需一个[Confluent Schema Registry](https://docs.confluent.io/current/schema-registry/docs/index.html) , 它用于保存客户端的所有schemas的历史版本，可以运行多个实例。其中一个Schema Registry实例被选举为主，负责注册新的schemas, 其余的作为从节点。从节点实例可以处理读请求并且将写请求转发到主节点。如果主节点有故障发生，则从节点重新选举出新的主节点，看起来它可以用一个Zookeeper或etcd集群来实现都没有问题（译者添加）。

上面这些合在一起，为单数据中心设计方案提供了在broker故障的情况下的强大保护。更多的如何配置和监控kafka集群消息持久化和高可用的信息，可详见[Optimizing Your Apache Kafka Deployment](https://www.confluent.io/white-paper/optimizing-your-apache-kafka-deployment/) 白皮书。

#### 多数据中心
在多数据中心的设计中，有多个或更多的数据中心中部署有Kafka集群。虽然多数据中心Kafka集群的方式有多种，但我们在这份白皮书里只关注于两个数据中心的灾难恢复。

考虑两个Kafka集群，每一个都部署在地理位置独立的不同的数据中心中。它们中的一个或两个可以部署在Confluent Cloud上或者是部分桥接到cloud。每个数据中心都有自已的一套组件：
* Kafka 集群， 在本地数据中心中的所有broker构成一个集群，完全不依赖远端数据中心中的broker;
* Zookeeper集群仅服务于本地的集群;
* 客户端仅连接到本地集群。

<div align=center>![kafka multicenter](file:///home/lw/图片/kafka-multicenter.png)</div>

在多数据中心设计中，其目的是为了跨区域同步数据。Confluent Replicator是Confluent Platform的高级功能，是这种设计的关键所在。Replicator从其中的一个集群中读取数据，然后将消息完整地写入到另一个集群，并且提供了一个跨数据中心复制的中心配置。新的Topic可以自动被感知并复制到目标集群。如果吞吐量增加，这个Replicator将自动扩容以适应这个增加的负载。

这个Replicator可以应用在多种不同的用户场景，这里我们关注它在两个Kafka集群作灾难恢复时的使用。如果一个数据中心发生部分或彻底的灾难，那么应用程序将能够故障转移到另一个数据中心。

在下面的主-从设计中，Replicator运行在一侧（通过应该是运行在目标集群一侧），从主集群DC-1拷贝数据和配置到从集群DC-2。

<div align=center>![kafka multicenter](file:///home/lw/图片/kafka-.png)</div>

生产者只写数据到主集群。依赖于整体的架构，消费者仅从主集群来读取数据，而从集群仅仅是作为灾难恢复用。当然，消费者同样也可以从两个集群都读取数据，用于创建基于本地地理位置的本地缓存。在稳定状态下，当两个数据中心正常运行时，DC-1是主集群，因此所有生产者只写入数据到DC-1。这是一种有效的策略，但对从集群的资源利用不够高效。如果灾难事件发生导致DC-1故障，企业需要确定客户端应用程序将如何响应。客户端应用程序可以故障转移到DC-2。当DC-1恢复后，作为故障恢复过程的一部分，DC-2中所有的最终状态信息也要复制回之前的主集群。

在下面的主-主（多主）设计中，部署两个Replicator, 一个将数据和配置从DC-1复制到DC-2, 另一个从DC-2复制到DC-1。

<div align=center>![kafka multicenter](file:///home/lw/图片/kafka-multi-replicator.png)</div>

生产者可以写数据到两个集群，DC-1的生产者写数据到本地DC-1的topic中，DC-2的生产者写数据到本地DC-2的topic中。DC-1的消费者可以消费本地DC-1的生产者生产的数据，也可以消费从DC-2中同步过来的数据，反之亦然。消费者能够通过具体的topic名字和统配符来订阅多个topic。因此，两个数据中心的资源都得到了很多的利用。如果灾难事件导致DC-1故障，已经存在的DC-2的生产者和消费者将继续它们的操作，它本质上不受影响。当DC-1恢复后，作为故障恢复过程的一部分，客户端应用程序可以直接回到之前的主集群。

#### 数据和Metadata的复制
单Kafka集群内部的数据复制是同步进行的，这意味着在数据被复制到本地的其他broker后，生产者才会收到ack。与此同时，Replicator在两个数据中心之间异步复制数据。这意味着生产数据到本地集群的客户端应用不会等待数据复制到远端集群就可以收到ack。这个异步复制使得对消费者消费到有效数据的延迟最小化。异步复制的另一个好处是你不用在两个不同集群之间创建相互依赖。即使两个集群之间的连接失败或者你需要维护远端数据中心，生产者发送数据到本地集群仍将是成功的。

Replicator复制的不仅仅是topic的数据还有metadata。比如topic metadata或者partition个数在原集群发生变化，Replicator同样可以将这种变化同步到目标集群。为了维护kafka topic的配置选项在多个集群一致，topic metadata必须在原始集群和目标集群保持相同。这个是由Replicator自动完成的。在topic准备阶段，它创建一个初始的topic配置，然后它在两个集群间现步这个topic的metadata。比如，如果你在DC-1中更新了一个topic的配置属性，Replicator将相应的配置更新到DC-2上对应的topic上。

#### MirrorMaker
你可以也听说过Kafka提供了一个单独的工具叫“MirrorMaker“，它可以用来在两个Kafka集群间复制数据。但是，MirrorMaker有很多的不足之处，使它在构建和维护多数据中心部署时面临很大的挑战，包括以下几点：
* 复制时为了过滤topic,需要繁琐的配置
* 在目标集群创建的topic使用的配置可能和原始集群不匹配
* 缺少内建的重新配置topic名字来避免循环复制数据的能力
* 没有能力根据kafka流量增加来自动扩容
* 不能监控端到端的跨集群延迟

Confluent Replicator解决了上面这些问题，提供了可靠的数据复制功能。Replicator提供了更好的跨多个数据中心数据和metadata同步的功能。它整合了Kafka Connect,提供了优秀的可用性和伸缩性。

另外，Confluent Control Center还可以管理和监控Replicator的性能，吞吐量和延迟。为了监控Replicator的性能，你需要配置 [Confluent Monitoring interceptors](https://docs.confluent.io/current/control-center/docs/installation/clients.html)

<div align=center>![kafka multicenter](file:///home/lw/图片/kafka-monitor.png)</div>

#### 中心化的Schema管理
>译者注： 
我们先简单过一个Schema是什么，它其实就是描述了消息的格式，比如一个消息体有什么字段，是什么类型等，在生产者和消费者之前达到一种消息格式的协议。Schema管理简单说就是有个中心服务，来管理全局的这些Schema,新的schema注册到Schema管理服务后，获取到一个唯一schema id,然后在生产的消息中带上这个schema id, 消息者获取到消息后，先解析出schema id，然后去schema管理服务上再获取到对就在的schema, 用这个schema到消息的具体内容解析出来。这个Schema管理服务通常是CP系统，分布式部署，数据强一致。Confluent提供了这样的一个服务，详情请见 [Schema Registry](https://docs.confluent.io/current/schema-registry/docs/index.html)

由于存储在Kafka的消息需要跨所有集群生产和消费，因此Schemas需要全局有效。Schema Registry提供了中心化的schema管理并且它被设计成分布式的。多个Schema Registry实例跨数据中心部署，提供了弹性和高可用性，并且任何的一个实例都可以将schemas和schema id发送到Kafka客户端。为了写kafka topic而用到的所有schema信息，都作为log提供到database(类似于zookeeper, etcd等)。在单主架构中，仅仅主Schema Registry实例可以写针对kafka topic的新的注册信息，从schema registry将新的注册请求转发给主。

在多数据中心的设计中，有多个数据中心的所有Schema Registry实例都应满足下列操作的要求：
* **访问相同的schema id**
由于DC-1中生产的消息可能需要在DC-2中被消费，因此Schema信息在多个数据中心中必须都是有效的。DC-1中的一个生产者注册新的schema到Schema Registry并且插入schema id到消息中，然后DC-2或任意一个数据中心中的一个消费者都可以使用这个Schema id从shema registry处查询到对就应的schema信息。

* **协调主schema registry的选举**
不论你的多数据中心是双主还是方从模式，都需要选定一个kafka集群威群作为主Schema Registry。这个集群将从Schema Registry所有实例中选出主。在Confluent Platform 4.0版本之后，kafka Group协议和Zookeeper都可以协调这个选主过程。如果连接到Confluent云或者是无法访问Zookeeper, 则可以使用kafka Group协议。

<div align=center>![kafka multicenter](file:///home/lw/图片/ele.png)</div>

Replicator也负现将保存有Schema 信息的kafka topic数据从主cluster同步到从cluster。多个数据中心中的所有从Schema Registry都直接订阅当前集群中的schema topic。需要时刻确保主Kafka集群和备选的Schema Registry实例在所有数据中心中都是合局可访问的。

## 关键特性
#### 避免消息的循环复制
在双主的情况下，需要在跨集群双向复制数据，那避免消息的循环复制就变得很重要。没人愿意看到topic的消息从DC-1复制到DC-2, 又从DC-2又复制回DC-1。

Conflument Replicator 5.0.1版本引入了新的特性，可以在不强制使用唯一topic名字的前提下，避名消息的循环复制。如果这个特性被开启，Replicator将针对每个消息都跟踪消息的来源信息，包括集群和原始topic。Replicator使用Kafka header这个新特性来跟踪来源信信息。Kafka header是在kafka 0.11及以上版本中支持，相应的broker的配置参数为`log.message.format.version`, 在kafka 2.0版本它是默认被设置的。

为了开启Replicator这个特性，需要配置`provenance.header.enable=true`。Replicator将放置跟踪信息到被复制后的消息的header中。这个跟踪信息包括下列部分：
* 消息被首先生产的初始集群的ID
* 消息被首先生产的初始topic名字
* Replicator首次复制该消息时的时间戳

默认情况下，如果目标集群topic名字和来源信息中的topic名字相匹配，并且目标集群ID和在这个来源信息header中的集群ID匹配时，Replicator将不复制消息。考虑下面这张图，在原始集群和目标集群中使用完全相同名字的topic,消息m1由初始的DC-1产生，消息m2由初始的DC-2产生。

<div align=center>![kafka multicenter](file:///home/lw/图片/d12png.png)</div>

当Repicator将消息从DC-1中复制到DC-2时：
* m1 将被复制到DC-2, 因为在DC-1的m1消息的消息header中没有任何的追踪信息
* DC-1中的m2不会被再次复制回DC-2, 因为DC-1中的m2消息的消息header中已经标识出来它初始来自DC-2

通常情况下，当Replicator能够自动避免循环复制消息时，不同数据中心的应用程序可以使用完全相同的topic名字来访问topic。

客户端应用程序的设计需要考虑跨数据中心有相同topic名字时的影响。生产都不会等待消息被复制到远端集群的ACK，并且当消息在本地集群被提交后，Replicator会异步在两个数据中心间复制消息。如果每个数据中心的生产者都使用相同的topic名字，这在全局来说是无序的（即使只有一个集群，也是无序的）。如果在每个数据中心都使用相同的group id来消费相同的topic,稳定情况下每个数据中心的消息都将被重新处理一次。

#### 保留时间戳
在Kafka集群内部，Kafka cosumer会跟踪它们已消费的消息。为了在停止消费后的某一刻继续消费，Kafka使用offset来标识下一条将要被读取的消息。这个消费者的offset保存在一个叫`__consumer_offsets`的特定的kafka topic里。

在多数据中心的情况下，如果某个数据中心发生灾难，消费者将停止从这个集群消费数据，可能需要消费另一个集群的数据。理想情况是新的消费者从旧的消费者停止消费的位置开始继续消费。你可以会尝试使用Replicator来复制consumer offsets这个topic。但是相同的offset在两个不同的数据中心集群中指向的message可能不是同一条，在这种情部下复制consumer offsets这个topic就是不能正常工作的。这个场景包括下面各种情况：
* 在消息被同步前，由于retention polict或者是compaction策略，原集群中的一些数据可能被清理。这通常发生在消息已经写入原始集群很长时间后Replicator才启动。在这种情况下，offsets将不再匹配。
* 在从原始集群向目标集群复制数据时，可能会发生短暂的错误，这将导致Replicator重送数据，就可能导致数据重复。可能有重复消息的后果就是相同的offset可能不再对应相同的消息。
* 数据topic的同步可能会落后consumer offset这个topic的同步。因为consumer offset topic和data topic的同步是各自独立的，所以可能会遇到这个落后的问题。当consumer在新的集群重新启动时，可能它尝试读取的offset对应的消息还没有被复制过来。

基于上面这些可能的场景，应用程序不能使用consumer offsets来作为两个集群中相同消息的标识。实际上这个`__consumer_offsets`topic不会在也两个数据中心间被复制。当复制Data时，Replicator会保留消息中的时间戳。Kafka新版本在Message中增加了时间戳支持，并且增加了新的基于时间戳的索引，保存了时间戳到offset的关联。

下面这张图显示了`m1`这个消息被从DC-1复制到了DC-2,这个message在两个集群中的offset是不同的，但保留了相同的时间戳`t1`。

<div align=center>![kafka multicenter](file:///home/lw/图片/time.png)</div>

当Kafka broker在message中保存了时间戳后，consumer就重置message的消费位置到之前的某个时间点。

#### Consumer Offset的转换
###### 故障转移后从什么位置恢复消费
如果发生灾难，consumers必须重启已连接到新的数据中心，并且它必须从灾难发生之前在原有数据中心消费到的topic消息的位置开始继续消息。

<div align=center>![kafka multicenter](file:///home/lw/图片/12.png)</div>

故障转移到另一个数据中心的consumers如何确定从这个topic的什么位置开始消费呢？可以从每个topic的最旧或最新位置开始消费。

考虑下面的情景，一个生产者写了10000条消息到DC-1中的一个topoic。Replicator复制这些消息到DC-2中。由于存在复制落后的可能，当灾难恢复时，它只复制了9998条数据。在灾难发生前，原有的consumers在DC-1中只读取了8000条数据，还剩下2000条没有读。故障转移到DC-2后，这个consumer看到这个topic有9998条消息，还有1998条没有读。那么它将从什么位置开始读取呢？

<div align=center>![kafka multicenter](file:///home/lw/图片/13.png)</div>

默认情况下，当一个consumer在DC-2创建后，这个配置参数`auto.offset.reset`将被设置为`latest`或`earliest`,如果设置为`latest`, 将从最新位置开始消费，将丢失掉1998条数据;如果设置成`earliest`, 会重复消费8000条数据。

有些应用可以接受从最新或最旧开始消费。但是，有些应用这两种方式都不能接受，它们期望的行为是从第8000条消息开始消费且仅消费1998条数据，就像下面这张图显示的。

<div align=center>![kafka multicenter](file:///home/lw/图片/14.png)</div>

为了满足这个需求，consumer需要在新集群中将conusmer offset重置到某些有意义的点。就像在`保留时间戳`这一节讨论的，consumers不能完全依靠offsets来重置消费的offset,因为这个offset在两个集群之间标识的消息可能是不同的。Offsets在两个数据中心间可能不同，但时间戳是一致的。在消息中保留的时间戳，在两个集群间有相同的意义，并且可以将这个时间戳对应的消息的offset作为开始消费的位置。

Confluent Platform 5.0版本引入了一个新的特性，可以使用时间戳自动转换offsets,因此consumers能够在故障转移到新的数据中心后，从原始集群中记录的消费位置开始继续消费。为了使用这个能务，需要使用一个叫[Consumer Timestamps Interceptor](https://docs.confluent.io/current/multi-dc-replicator/replicator-failover.html#configuring-the-consumer-for-failover)的拦截器来配置java消费程序，它会保存已消费的消息对应的metadata信息，包括：
* Consumer group ID
* Topic名字
* Partiton
* 已提交的offset
* 已提交的offset对应的时间戳

这个Consumer的时间戳信息是保存在原始kafka集群中一个叫`__consumer_timestamps`的topic里。Replicator不会复制这个topic,因为它只有本地的集群中有意义。

Confluent Replicator将数据从一个数据中心复制到另一个的同时，还并行地完成下面的工作：
* 从原始集群的`__consumer_timestamps`topic中读取consumer offset和对应的时间戳信息来了解当前这个consumer group的消费进度
* 转换这个原始集群中的提交的offset到目标集群中对应的offset
* 只要没有这个group中的consumer边接到这个目标集群，就将转换得到的offset写入到目标集`__consuer_offsets`topic中

当Replicator将转换后的offset写入到目标集群的`__consumer_offsets`topic时，它需要知道每个offset对应的topic名字，这个topic名字按照`topic.rename.format`的配置被重命名。

如果已经有相应的consumer group中的consumer连接到了目标集群，Replicator将不会写入offset到这个`__consumer_offsets`topic。不论是哪些方案，当一个消费者故障转移到备份集群时，它将使用正常的机制查看并找到先前提交的offsets。

###### 转换后的Offset的准确度
使用上一节中介绍的Consumer时间戳拦截器，故障转移到新数据中心后的conusmer group就可以从故障的集群中已提交的offset的位置开始消费了。消费都将不会少消费任何的消息。但是依赖于影响转换offset的若干因素，消费者可能会重复消费一些消息。

影响转换offset的若干因素有：
* 复制的落后情况
* offset的提交周期
* 有相同时间戳的记录的数量

## 多数据中心部署
#### Confuent Replicator
###### 配置Repicator	
Confluent Replicator是一个Kafka connector,它运行在Kafka Connect框架内。Replicator继承了所有Kafka Connect API的优点为，包括伸缩性，性能和容错。Confluent Replicator从原始集群消费消息然后将消息写入到目标集群。这个Kafka Connect workers部署在和目标集群相同的数据中心。

下图显示了在主-从设计中这个Replicator connector运行在DC-2中，将数据单向从DC-1复制到DC-2。
<div align=center>![kafka multicenter](file:///home/lw/图片/112.png)</div>

在双主设计中，数据是被双向复制的。一个Replicator connector将数据从DC-1复制到DC-2，另一个将数据从DC-2复制到DC-1。如下图所示：
<div align=center>![kafka multicenter](file:///home/lw/图片/114.png)</div>

这两个Replictor connector的配置参数是不同，下表是它们的配置：

<div align=center>![kafka multicenter](file:///home/lw/图片/115.png)</div>

值得注意的是， Confluent Replicator自己内置了一个consumer, 默认情部下，会提交自己的consumer timestamps到原始集群。它的offsets会被转换到目标集群。在主从设计中，DC-1宕机过程中在DC-2中产生的新消息，在DC-1又恢复后需要被复制回DC-1, 这个Replicator就被用来恢复服务。然而，在双主设计中，它是不需要的，因为有两个独立的Replicator运行在双向上。为了简化双主设计的工作，我们会通过配置` offset.timestamps.commit=false`来禁止Replicator提交自己的consumer timestamps。

在双主设计中，这里有一个最小化的Replicator配置，用于从DC-1到DC-2的复制。我们假设有一个topic白名单，里面有一个topic `topic1`, 它已经在两个数据中心都配置好。如果你使用Confluent Schema Registry，这个topic 过滤器还应该包括这个topic `_schemas`,但它只需要单向复制。下面的配置允许Replicator使用message header信息来避免topic的循环复制。

<div align=center>![kafka multicenter](file:///home/lw/图片/116.png)</div>

这里有一个最小化的Replicator配置，用于从DC-2到DC-1的复制, 它和上面的配置是很相似的，除了它有不同的名字和consumer group ID, 还有它交换了` src.kafka.bootstrap.servers`和` dest.kafka.bootstrap.
servers.`的值，并且它不需要复制`_schemas`。

<div align=center>![kafka multicenter](file:///home/lw/图片/117.png)</div>

###### 运行Replicator
这一节描述了在Kafka Connect集群内部如何将Replicator作为不同的connector来运行。为了在Kafka Connect集群里运行Replicator,你首先需要初始化Kafka Connect,在生产环境为了伸缩性和容错将总是使用分布式模式。你可以使用[ Confluent Control Center](https://www.confluent.io/confluent-control-center/)来作所有Kafka connectors的集中式管理。

<div align=center>![kafka multicenter](file:///home/lw/图片/118.png)</div>

然后将Replicator connector添加到Kafka Connect。这个Repicator将运行在DC-2中将数据从DC-1拷贝到DC-2，另一个Replicator运行在DC-1中将数据从DC-2拷贝到DC-1。

<div align=center>![kafka multicenter](file:///home/lw/图片/119.png)</div>

你可以通过REST API和Kafka Connect交互，来管理和检查这些connector:

<div align=center>![kafka multicenter](file:///home/lw/图片/120.png)</div>

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

<div align=center>![kafka multicenter](file:///home/lw/图片/121.png)</div>

#### Confluent Schema 注册表
Confluent Schema注册表提供了丰富的配置项，这份白皮书只讲解几个重要的配置。首先，为每个Schema Registry实例配置一个唯一的`host.name`。我们需要改变这个参数的默认值`localhost`。这是因为其他的`Schema Registry`必须能够解析这个hostname,你也需要能够追踪哪个实例当前是主。

生产者和消费者客户端需要使用一个一致的schema ID来源，通常使用主数据中心的一个Kafka topic来作为这个来源，其topic名字通过`Kafkastore.topic`这个参数来指定。当第一个Schema Registry实例启动时，这个topic被自动创建，它有很高的持久化设置，据有三个复本，清理策略是压缩。

通过`kafkastore.bootstrap.servers`来配置所有的Schema Registry实例都使用DC-1这个Kafka 集群。需要复制schema topic到作为备份，这个在DC-1发生灾难时，你仍然可以反序列化消息。

最后，在主数据中心中配置所有的Schema Registry实例都可以参与选举成为主，他们将允许注册新的schema，配置第三个数据中心中的所有Schema Registry实例不能参与选主，禁止通过它们来注册新的schema。

<div align=center>![kafka multicenter](file:///home/lw/图片/022.png)</div>

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

<div align=center>![kafka multicenter](file:///home/lw/图片/023.png)</div>

考虑DC-1发生灾难事件时的故障转移流程。

首先，当DC-1发生故障时，客户端应用程序连接到DC-1将超时或完全失败，它们将连接到DC-2来继续生产和消费。DC-2中的Replicator也将继续运行，但因为DC-1已经故障，它将不再能从DC-1中消费到数据。

由于存在复制落后的情况，有些已经写入到原始集的消息还没有来得及到达目标集群。考虑这样一个例子，生产者将1000条消息写入到了DC-1的topic。Replicator异步将这些消息复制到DC-2，但是当灾难事件发生时，最后有一些消息没有被复制到DC-2,比如说在灾难事件发生前，只有9998条消息被复制了，这个结果就是丢失了一些消息。

<div align=center>![kafka multicenter](file:///home/lw/图片/024.png)</div>

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

<div align=center>![kafka multicenter](file:///home/lw/图片/025.png)</div>

## 故障恢复
#### 恢复Kafka集群
当原来故障的集群从灾难事件中恢复后，你需要恢复多数据中心的配置，在两个kafka集群间同步数据并且正确地重启客户端应用程序。这个故障恢复流量看起来似乎只需要处理新数据。但实际上Kafka的故障恢复需要进一步的考虑，因为Kafka需要复制已经消费的数据和还未消费的数据。

<div align=center>![kafka multicenter](file:///home/lw/图片/026.png)</div>

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

###### 客户端应用程序重启
一旦故障恢复，数据同步完成，客户端应用程序将要切换完原始主集群，需要重新初始化来连接到原始主集群。

## 总结
这份白皮书讨论了架构，配置等构建模块和后续的故障转移，故障恢复流程。这里涉及了多种针对多数据中心架构的用户场景，但焦点集中在灾难恢复上。你的架构将非常依赖于你的业务需求。你需要应用合适的构建模块来增加你的灾难恢复预案。