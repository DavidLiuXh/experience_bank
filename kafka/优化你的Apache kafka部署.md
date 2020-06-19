[toc]

# 优化你的Apache Kafka部署

> 翻译自 https://www.confluent.io/wp-content/uploads/Optimizing-Your-Apache-Kafka-Deployment-1.pdf

## 前言

Apache kafka是一套可以拿过来直接运行起来的很好的企业级流处理平台。只需要将你的客户端应用放到Kafka集群中，剩下的事件就都可以交给Kafka来处理，比如：负载在brokers之间的自动分布，brokers自动借助零拷贝传输技术发送数据到消费者，当有消费者加入或离开时consumer groups自动均衡，应用程序使用Kafka Streams APIs将状态存储自动备份到集群中，当broker故障时partition主自动重新选举。这样看起来，运维人员的梦想成真啦！

在不需要对Kafka配置参数作任何改动的情况下，你就可以部署起来一套Kafka的开发环境并且测试基本功能。但事实上Kafka可以直接运行起来并不意味着在上到生产环境前你不需要作一些调整。需要作调整的原因是，不同的用户场景有不同的需求集群，这将最终驱动不同的服务目标。为了针对这些服务目标来作优化，你将需要改变Kafka的某些配置参数。实际上，Kafka自动的设计就给用户提供了灵活的配置。为了确保你的Kafka环境是针对你的服务目标作了优化的，你必须要调整一些配置参数的设定并且在你的环境中作基准测试。理想情况下，你将在上到生产环境前完成这些，或者至少在将集群规模扩充到比较大之前完成。

这份白皮书涉及到如果确定你的服务目标，配置你的Kafka部署来优化它们，通过监控来确保达到了你的目标。

<div align=center>![tab block](file:///home/lw/图片/327.png)</div>

## 确定针对哪些服务目标作优化

第一步是先确定你希望针对哪些服务目标作优化。我们认为经常需要在四个目标间作相互权衡：吞吐量，延迟，持久化和可用性。为了发现你需要优化的目标，回想你的集群想要给哪些用户场景打供服务。思考一下应用和业务需求--针对这些用户场景作到绝对不能失败将是最满意的结果。思考一下Kafka作了一个流处理平台是如何填充你的业务管道的。

<div align=center>![tab block](file:///home/lw/图片/328.png)</div>

有时确定需要优化的服务目标是一个很难回答的问题，但是你必须强制你的团队讨论并确定你们最基本的业务使用场景和主要目标是什么。对这个讨论而言有两个原因是很重要的。

首先一个原因是你不可能在同一时间将所有的目标都最大化。它需要在吞吐量，延迟，持久化和可用性间作权衡，我们将在这份白皮书中详细阐述这些服务目标。你可能熟悉常见的在吞吐量和延迟间的性能权衡，也可能熟悉在持久性和可用性之间的权衡。从整体上来考虑，你会发现你不能孤立地来考虑它们，这份白皮书就是将他们放在一起来考虑。这不是说我们对目标中的一个作优化而完全丢掉其他的。它仅仅意味着这些服务目标都是有内在联系的，但你不可能在同一时间内对所有的都作出优化。

确定对哪些服务目标作优化的第二个重要原因是你能够并且也可以通过调整Kafka配置参数到达成它。你需要明白你的用户期望从系统中得到什么来确保你优化Kafka来完成他们需要的。

* 你希望针对高吞吐量，即数据生产或消费的速度，来作出优化吗？有些使用场景每秒钟可以写入上百万条消息。基于Kafka本身的设计，写入大量的数据对它来说不是难事。它比写入大量数据到传统数据库或key-value存储要愉，并且它可以使用先进的硬件来完成这些操作。

* 你希望针对低延迟，即消息在端到端到达上的时间间隔，来作出优化吗？ 低延迟的一个使用场景是聊天应用，它总是希望消息接收者越快收到消息越好。另外一些例子包括交互式web应有，比如用户看到他朋友们的更新，或者是物联网中的实时流处理。

* 你希望对可靠的持久性，即保证消息被提交后将不会丢失，来作出优化吗？ 可靠持久性的一个使用场景是使用kafka作为事件存储的事件驱动的微服务管道。另一个例子是为了交付关键业务内容而整合数据流来源和一些永久存储（比如 AWS的S3）。

如果你希望优化的服务目标需要覆盖Kafka集群中的所有topic,那么你可以在所有brokers上设置相应的broker级别的配置参数来将期应用到全部的topic上。换句话说，如果你想针对也不同的topic作不同的优化，你同样可以重写对应的topic的配置参数。Topic没有明确地被重写了配置的，将应用这个broker的配置。

Kafka有上百个不同的配置参数，这份白皮书只会针对我们的讨论中用到的一部分配置。这些参数的名字，描述和默认值在Confluent Platform version 3.2中已经更新到最新。关于这些配置参数，topic复写和其他一些参数的更多信息可以在[http://docs.confluent.io](http://docs.confluent.io.)找到。

在我们针对不同的服务目标作优化前有一点需要注意：我们在这份白皮书中讨论的一些配置参数的值取决于另一些因素，比如消息的平均大小，partition个数等。它们可能因环境不同而有很大不同。对于一个配置参数，我们提供了配置值的一个合理的范围，回想一下，基准测试总是能够很多地验证我们针对特定部署而作的设置。

#### 优化吞吐量

<div align=center>![tab block](file:///home/lw/图片/329.png)</div>

为了优化吞吐量，生产者，消费者和brokers都需要在给定的时间内移动尽可能多的数据。对于高吞量，你需要尝试将数据移动的速度最大化。这个数据移动的速度越快越来。

Topic的partition是Kafka系统中最小的并发单元。生产者可以并行地将消息发送到不同的partition,并行地写入到不同的brokers上，消费者也可以并行地从不同的partition上消费数据。通常来讲，数量多的topic paritions会带来高的吞吐量。然后创建据有大量partition的topic看起来很有诱惑力，但依然需要权衡这种作法。我们需要基于生产者和消费者的吞吐量来小心地确定partition的数量。这里有篇文章[How to choose the number of topics/partitions in a Kafka cluster?](https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster) 专门来讲解了为何选择partition数量并需要在你的环境中确认基准性能测试。

接下来我们讨论一下kafka生产者的批量发送策略。生产者能够将消息批量发送到同一个partition, 也就是说将多个消息收集到一个发送请求中然后一起发送出去。我们优化吞吐量很重要的一步就是调整这个生产者批量发送的参数，包括增加批量发送的大小和等待添充满批量发送队列所耗费的时间。大的批量发送大小使得只有很少的请求发送到brokers，这降低了在生产者和brokers上处理每条请求的的CPU负载。在Java客户端中，可以配置`batch.size`参数来每次批量发送的最大字节数。为了能有更多的时候来添充批量发送的队列，你可以配置参数`linger.ms`来让生产者在发送前等待更长的时间。这需要权衡一下是否能容忍高的延迟，因为在这种情况下消息不是在准备好之后就立即发送。

你同样可以通过配置`compression.type`参数来开启压缩功能。压缩意味着按照压缩算法的使用，大数据量可以变成小数据量被发送。Kafka支持lz4, snappy和gzip压缩算法。压缩算法可以应用到每个完整的数据batche上，这样可以更好地提高压缩比。

当生产者发送消息到Kafka集群集地，这条消息是被发送到目标partition的主所在的broker上。在发送下一条消息前，生产者总是要一直等待leader broker的响应来知晓这条消息是否已经提交。服务端是自动检测以确保消费者不能读取未提交的消息。leader brokers何时发送响应可难会影响到生产者的吞吐量：生产者越早地收到响应，就能越早地发送下一条消息，这通常也会产生高的吞吐量。生产者可以设置配置参数`acks`来指定leader broker在发送给生产者ack响应前需要收到多少个followers的ack。如果设置`acls=1`，则leader broker在将消息写到本地的log中后，不用接收到任何一个followers的ack就能够发送响应到生产者。这需要权衡你是否能容忍低的持久性，因为这时生产者不能够等待到消息被复制到其他的brokers。

如果发送失败，生产者可以重试，直到重试次数达到配置参数`retries`指定的上限。如果应用程序能够处理一些数据的丢失，你可以设置`retries=0`。在这种情况下，如果消息发送失败，生产者将不会尝试重新发送这个相同的消息并且消耗的带宽将分配给其他的消息。

对于Java客户端，Kafka生产者可能自动分配内存来存储未发送的消息。如果内存的使用达到上限，生产者会阻塞额外的消息发送直到内存释放或者直到`max.block.ms`时间过去。你可以通过配置参数`buffer.memory`来调整分配置内存的大小。如果你没有大量的partitions,你可能不需要调整这个大小。然而，如果你有大量的partition,你需要综合buffer size, linger time和partition数量来调整这个参数。通过调整这个参数，使得生产者在阻塞额外的消息发送前将经历很长的时间，这样也就提高了吞吐量。

同理，你也能够通过调整消费者每次从leader broker拉取的数据量的大小来提高吞吐量，它可以通过调整`fetch.min.bytes`这个参数来完成。增加这个参数会减少发送到broker的fetch请求的数据，降低broker处理每条fetch请求的CPU负载，这将改善吞吐量。与在生产者上增加batch大小相似，当增加这个参数时，需要考虑高延迟的权衡。这是因为broker不会立即发送新的消息到消费者，直到有足够的消息来填满`fetch.min.bytes`，或者直到等待时间过期。

假设应用程序允许，可以使用由多个消费者组成的消费组来并行消费。并行消费可以提高吞吐量，因为多个消费者可以自动作负载均衡，同时消费不同的partition。

最后，你可以调整JVM参数来最小化GC的停止时间。GC对于删除不再使用的对象和回收内存是很有必要的。但是，长的GC停止间对提高吞吐量有负面影响，在最坏情况下还会导致broker的软故障，比如zookeeper会话超时。

###### 优化吞吐量的一些配置
生产者:
* batch.size: increase to 100000 - 200000 (default 16384)
* linger.ms: increase to 10 - 100 (default 0)
* compression.type=lz4 (default none)
* acks=1 (default 1)
* retries=0 (default 0)
* buffer.memory: increase if there are a lot of partitions (default 33554432)

消费者:
* fetch.min.bytes: increase to ~100000 (default 1)

#### 优化延迟
在上面吞吐量一节中讨论的有些Kafka配置参数有些默认值也可以优化延迟。虽然这些参数通常不需要调整，但我们还是需要弄清楚它们是如何工作的。

Partition是Kafka的最小并行弹单元，增加Partition数量可以增加吞吐量。但是，需要权衡在增加Partition数量的同时会增大延迟。默认情况下一个Broker使用单一的一个线程从其他broker来复制数据，因此在两两broker之间复制大量Partition将花费更长的时间并且这样的结果是消息被最终认定为提交状态也需要花费更长的时间。没有提交的消息不能被消费，因此这又增加了端到端的延迟。

为了解决这个特定的拉取消息的延迟，一个选择是尝试限制任意一台broker上的partition数量。你可以通过限制整个集群的partition数量或者增加集数的broker数量来达到这个目的。对于需要低延迟但又需要大量partiton的实时处理类应用程序，你可以调整同步线程个数。这可以通过调整`num.replicat.fetchers`参数来完成，它会增加在follower broker上的并行IO资源。你可以在它是默认值1的时候作基准测试，如果follwers不能跟上leader的数据写主，你可以增加大这个配置。

你可以考虑是否需要开启压缩功能。开启压缩通常需要更多的CPU周期，但可以减少网络带宽的占用。反之，会增加网络带宽占用。好的压缩编码方式也可能潜在地降低延迟。

你可以调整在认为发送请求完成前生产者需要leader broker收到的ACK数量。leader broker越快的响应，生产者也就能更快地继续发送下一条消息，这通常可以降低延迟。可以通过生产者的`ack`配置参数来设置所需要的ACK的数量。默认`acks=1`, 意味着只要leader broker在消息写入了本地存储，在所有复本收到这个消息之前，就可以响应生产者了。依赖于你的应用的需求，你可以设置`acks=0`, 这样会让生产者不等待broker的任何响应就可以发送下一条消息，但这样潜在地可能会丢失数据。

与生产者批量发送的目的类似，你也可以调整消费者每次从leader broker拉取的数据量的大小来优化这个延迟。默认情况下，消息者配置参数`fetch.min.bytes=1`，它意味着只要一字节的数据是有效的，fetch 请求就会返回，或者是在有效数据到达前fetch reqeust超时了。

有些场景，你需要执行大规模，低延迟地table 查询操作，你可以使用Kafka Stream API来作本地的流式处理。一个流行的方案是使用Kafka Connect将远程数据库存的数据拉取到本地的kafka系统中，然后你就可以利用Streams API来执行特别快速和有效地一些tables的本地的join操作和流处理，而不再需要应用程序针对每条记录都参通过网络发起一次针对远程数据库的查询操作。你可以跟踪在本地状态存储中的每个table的最新状态，当你作类似于streaming joins这些的操作时，将极大地降低处理的延迟。

###### 优化延迟的一些配置
生产者：
* linger.ms=0 (default 0)
* compression.type=none (default none)
* acks=1 (default 1)

Broker:
* num.replica.fetchers: 如果follwers不能跟上leader的数据写主，你可以增加大这个配置(default 1)

消息者:
* fetch.min.bytes=1 (default 1)

#### 优化持久化存储
持久化是降低消息丢失的全部机会之所在。开启持久化的最重要的特性是复制，它意味着消息将被拷贝到多个broker上。如果一个broker故障，数据还可以在至少一个broker上是有效的。对有高持久化需求的topic来说，需要将配置参数`replication.factor`设置为3，这将确保集群在坏掉两台broker时也不丢失数据。如果Kafka集群开启了topic自动创建功能，那么你需要考虑改变配置参数`default.replication.factor`到3，使自动创建的topic也有复本，或者禁止topic自动创建，由你自己来控制每个topic的复本数和partition设置。

复本对于被客户端使用的所有topic的持久化来说是很重要的，对于像`__consumer_offsets`这种Kafka内部topic来说也是很重要的。这个topic跟踪已经被消费的消息的offsets。除非你运行的kafka版强制为每个topic设置复本，那你应该小心处理topic的自动创建。当启动一个新集群时，在开始从topics消费数据前，应当至少等待三个brokers在线。这可以避免自动创建的topic`__consumer_offsets`的复本数比配置参数`offsets.topic.replication.factor`定义的复本数还少。

生产者可以通过`acks`配置参数来控制写到Kafka的消息的持久性。这个参数在吞吐量和延迟优化中讨论过，但是它主要是用在持久化方面。为了优化高的持久性，我们建议设置它为`acks=all`，这意味着leader将等待收到in-sync列表中所有复本的ack回应后才认为这个消息被提交了。它强有力地保证了只要in-sync列表中的复本只要有一个还活着，数据就不会丢失。这需要权衡是否能容忍高的延迟，因为在响应生产者之前，leader需要等待所有in-sync表表中复本的回应。

生产者同样也可以通过在发送失败时尝试重新发送的方式来增强持久性。这可以自动完成也可以手动完成。生产者自动重试的次数上限是通过`retries`参数指定的。生产者手动重试是依赖于返回给客户端的异常来完成的。如果你希望生产者自动处理重试操作，你可以设置`retries`为大于0的数。如果你配置了`retries`，有两件事情你需要考虑：
 1. 如果集群有一个瞬时的抖动，可以造成消息重复。为了处理这种情况，你需要确保你的消费者能够处理重复消息。当前，我们正致力于开发消费且仅消费一次语义的支持，它将帮助解决这个问题。
 2. 多次发送尝试可能是在相同时间内并且可能发生在另一个成功发送之后，这可能导致发送乱序。为了在允许重发失败的消息的前提下也保持消息顺序，你需要设置配置参数`max.in.flight.requests.pre.connection`为1来确保同一时间仅有一个请求发送到broker。

Kafka集群通过在多个brokers间复制数据来提供持久性。每个Paritition都有一个需要复制其数据的复本的列表，能够跟上leader数据写入进程的复本列表被叫作`in sync replicas`（ISR）。对于每一个partition， leader broker将自动复制消息到ISR列表中的其他broker，实际上是其他复本主动去leader broker上拉取。当生产者设置了`acks=all`时，然后这个配置参数`min.insync.replicas`可以针对ISR列表里复本个数指定一个最小的阈值。如果这个最小的复制数没有达到，生产者将产生一个异常。`min.insyc.replicas`和`acks`一起使用能使持久化得到强制保证。一个典型的场景是创建一个topic,它的`replication.factor=3`，broker的`min.insync.replicas=2`并且`acks=all`，这可以确保在大多数复本都没有接收到数据时，生产者将产生一个异常。

Kafka将针对broker故障提供的保障扩展到能覆盖机架故障。这个机架感知特性能够将相同partition的复本分散到不同的机架。这样就限制了全部broker都在一个机架上时机架发生故障导致的数据丢失。这个特性同样可以通过将broker分散到不同的有效区域的方式应用到像度亚马逊EC2这样的云解决方案上。你可以通过设置配置参数`broker.rack`来指定broker属于哪个机架，这样Kafka将自动确保复本尽可能多地分散到现不的机架。

如果有broker故障，Kafka集群能够自动侦测这个故障并且选举出新的partition主。新的partition主是从正在运行中的复本中选出。在ISR列表中的brokers都有最新的消息，并且它们中的一个将变为新的主，它能够从之前的主broker中断的位置继续拷贝消息到其他仍需要追赶的复本上。配置参数`unclean.leader.election.enable`标识在ISR列表中之外的没有追赶上leader的brokers是否能变为新的主。对于高持久性而言，需要通过设置`unclean.leader.election.enable=false`来确何新的leader仅仅从ISR列表中选举出来。这可以避免消息因已提交但没复制而丢失的风险。这需要权衡是否能容忍更多的不工作时间直到足够多的复本重新回到同步状态，即回到ISR列表中。

因此我们强烈建议你为了持久性而使用Kafka复本并且允许OS控制数据从page cache同步到磁盘，你通常不需要更改flush的设置。但是，对于对吞吐率要求极低的关键topics来说，在OS将数据flush到磁盘前可能有比较久的时间周期。对于这样的topics，你可以考虑调整`log.flush.interval.ms`或`log.flush.interval.messages`到比较小。例如，如果你需要将每条消息都实时持久化到磁盘，你可以设置`log.flush.interval.messages=1`。

你同样需要考虑如果消费者遇到不可预知的故障时如何确保再次处理消息时，消息不丢失。Consumer offset用来跟踪已经消费了的消息，因此消费者何时提交，如何提交message offset对于持久性来说就很关键。你肯定想避免这样的情况发生：消费者提交了消息的offset，然后开始处理这个消息，并且发生了不可预支的故障。这将导致后续从这个partition读取消息的消费都不能再重新处理这个消息，因为它的offset已经被提交过。你可以通过配置参数`auto.commit.enable`来配置offset的提交方式。默认情况下，offset被配置成在消费者周期性地调用`poll()`期间自动提交。但是如何消息者是事务链中的一部分，那么你需要消息到达的强保证。对于持久性来说，通过设置`auto.commit.enable=false`来禁用掉自动提交并且显式地调用`commitSync`或者`commitAsync`等提交方法。

###### 优化持久化的一些配置

生产者:
* replication.factor: 3, configure per topic
* acks=all (default 1)
* retries: 1 or more (default 0)
* max.in.flight.requests.per.connection=1 (default 5),避免消息乱序

Broker:
* default.replication.factor=3 (default 1)
* auto.create.topics.enable=false (default true)
* min.insync.replicas=2 (default 1); topic override available
* unclean.leader.election.enable=false (default true); topic override available
* broker.rack: rack of the broker (default null)
* log.flush.interval.messages, log.flush.interval.ms: 对于吞吐量很低的topics，将这两个参数设置低一些是需要的。

消费者:
* auto.commit.enable=false (default true)

#### 优化可用性

为了优化高可用性，你需要调整Kafka，以便其能够快速从故障中恢复。

很高的partiton数量可以增加并发，增加吞吐量，但它也可能会增加从broker故障事件中恢复所需的时间。所有的生产者和消费者都将暂停，直到主选举完成，并且每一个partition都需要进行leader选举。因此在选择partition数量时需要考虑故障恢复时间。

当一个生产者设置了`acks=all`，配置参数`min.insync.replicas`指定为认为写消息成功时需要回应ack的最小复本数。如果这个最小复本数都不能达成，生产者将产生一个异常。在ISR列表收缩的情况，生产者发送消息更可能失败，这将降低partition的高用性。换言之，设置它为一个比较低的值，比如`min.insync.replicas=1`，则系统将能容忍更多的复本故障。只要满足最小的复本个数，生产者发送消息将继续成功，这增加了partition的可用性。

Broker故障将导致partition选举，它会自动进行。你可以控制哪些broker有能力选举成为主。为了优化持久性，新的主仅从ISR列表中选举出来，这样作可以避免因消息提交了但没有复制而导致丢消息的风险。相比之比，为了优化多可用性，新主可以允许从ISR列表中移除的brokers中选举出来，这可以通过设置`unclean.leader.election.enable=true`来实现。它可以让leader的选举很快的发生，增强了整体的可用性。

当Broker启动时，为了和其他broker同步数据，它会扫描日志数据文件。这个过程被称作`日志恢复`。针对每个数据目录，在启动中用作恢复和在关闭时用作flush的线程数由配置参数`mun.recovery.threads.per.data.dir`来控制。有数千个log segments的brokers也会有大量的索引文件，它们会导致在broker启动时log的加载很慢。如果你使用RAID，那么你可以将`num.recovery.threads.per.data.dir`增加到磁盘个数大小，可能会减少日志加载时间。

最后，在消费者一侧，消费者作为消息组的一部分来共享处理所有的消费负载。如果一个消费者发生故障，Kafka能够侦测到错误并且对这个消费组中余下的消费者作负载均衡。通过配置参数`session.timeout.ms`来设置用来确定消费者故障所需的超时时间。消费者故障分为硬故障（比如 SIGKILL）和软故障（比如 session超时）。软故障的发生通常有两种情况：通过`poll()`返回的批量消息处理的时间过长，或者JVM GC停顿时间过长。如果session超时时间设置得较短，可以很快地侦测到消费者故障，这就减小从故障中恢复的时间。 

###### 优化可用性的一些配置
Broker:
* unclean.leader.election.enable=true (default true); topic override available
* min.insync.replicas=1 (default 1); topic override available
* num.recovery.threads.per.data.dir: number of directories in log.dirs (default 1)

Consumer:
* session.timeout.ms: 越低越快 (default 10000)

#### 基准测试，监控和调优

基准测试很重要，因为对于上面我们讨论的配置参数没有一种配置可以适用于所有的场景。合适的配置总是取决于使用场景，每台broker的硬件配置，你开启的一些特性和数据的配置等。如果你想要调整Kafka的默认配置，我们通常建议你作基准测试。不管你的服务目标是什么 ，你都需要明白这个集群的性能配置是什么--当你想优化吞吐量或延迟时，它特别重要。你的基准测试同时也可以使用计算并确定合适的partition数量，集群规模和生产者，消费者进程数量。

针对一个使用默认配置的测试环境来开始基准测试，并且熟悉这些默认值是什么。确定针对生产者的输入性能基线。首先需要从生产者上移除任何的上游数据依赖。不要从上游数据源接收数据，更改你的生产者以很高的输出频率来产生模拟数据，并且这些模拟数据的产生不要成为瓶颈。如果你测试时使用了压缩，你需要留意这些模拟数据的产生方式。有时产生的模拟数据都是被无意义的0所填充的。这可能会导致压缩性能要好于生产环境的性能。确保填充的数据能反映真实的生产环境的数据。

在单个server上运行单个生产者。Kafka集群有足够大的容量，因此它没有瓶颈。可以使用有效的JMX metrics来统计Kafka生产者的最终吞吐量。重复这个生产者的基准测试，在每次迭代中增加生产者进程数量，来确定一台server上达到最大吞吐量时的生产者进程数量。你可以用类似的方法来确定针对消费者的输出流量的性能。

接下来我们可以用能反应我们服务目标的配置参数的不同组合来运行基准测试。将配置参数集中在这个白皮书中讨论的，并且抵挡住发掘和更改那些你没有完全明白对整个系统有何影响的参数的默认值。集中在我们已经讨论过的参数，迭代地调整他们，运行测试，观察结果，再调整，直到这设置很好满足了你的吞吐量和延迟。

在你迁移到生产环境前，确保针对brokers, 生产者，消费者，topics和其他你使用的kafka组件都有一个强有力的监控。为了将内部的JMX metrics暴露给JMX工具，你在开始运行broker进程时，加上JMX_PORT这个参数。

在运维一个Kafka集群的监控系统时，你需要基于你的服务目标来考虑你的报警设置。这些报警可能是因topic而异的。有的topic可能有低延迟的需求，有的topic可能有高吞吐量的需求。