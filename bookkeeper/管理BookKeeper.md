[toc]



## ApacheBookKeeper 入门

参考 https://bookkeeper.apache.org

#### 设计理念和总体架构

首先会介绍下BookKeeper的核心组件和其是如何工作的。

BookKeeper服务提供了持久化存储log entries流的能力，这个log entries流称为`ledgers`，并且这个entries会以多复本的形式跨多个BookKeeper服务存储。

###### 名词解释

我们先就BookKeeper中的一个关键名词作下解释：

* entry: 可以简单认为是客户端写入的每一条消息
* ledgers：若干条entry有stream方式写进来构成一个 ledger;
* fragment: 每个ledger在磁盘上分片存储，划分成多个fragment;
* bookie: 一台独立的BookKeeper Server, 存储若干个ledgers;
* ensemble: 一个ledger的所有fragment所在的Bookie的集合;

我们用张图来具象化一下：

![](/home/lw/文档/basic terms of apache bookkeeper.png)

BookKeeper被设计成可靠的，能自动从多种故障中恢复，比如Bookies可能崩溃，数据错误或者数据在某一台bookies上丢人，但是只要有足够的行为正确的bookies存在于整个服务集合中，那其对外表现出的行为就是正确的。

###### Entries

Entries包含了实际写到ledgers的数据和与其一起的metadata，是字节序列。每个`Entry`有下面这些字段：

| Field               | Java type | Description                                                  |
| :------------------ | :-------- | :----------------------------------------------------------- |
| Ledger number       | `long`    | The ID of the ledger to which the entry has been written     |
| Entry number        | `long`    | The unique ID of the entry                                   |
| Last confirmed (LC) | `long`    | The ID of the last recorded entry                            |
| Data                | `byte[]`  | The entry’s data (written by the client application)         |
| Authentication code | `byte[]`  | The message auth code, which includes *all* other fields in the entry |

###### Ledgers

`Ledgers`是存储在BookKeeper中的基本单元。它是一系列`entries`的序列。`Entry`依照其`Ledger number`字段来写入到相应的`Ledgers`。这意味着`ledgers`是有*append-only*语意的。`Entries`一旦被写入ledger就不能被更改，并且由客户端应用来决定其写入顺序。

###### Clients and APIs

BookKeeper客户端有两种主要角色：它们负责创建和删除ledgers,并且也从ledgers读取数据或写入数据。它提供了低阶API和高阶API来和`ledgers`交互。

* 低阶的 ledger API 允许用户直接和`ledgers`交互；
* 高阶的 DistributedLog API 封装了ledgers的相关操作，用户不需要了解`ledgers`的细节也可以使用`BookKeeper`；

通常来说选择哪一种API取决于你是否需要直接操作`ledgers`。

###### Bookies

`Bookie`是独立的BookKeeper服务，它用来处理具体的`ledgers`的请求， `ledgers`在存储时实际上作了分片fragment, 因此Bookies实际上处理的是`ledgers`的fragment， 而不是整个`ledgers`。比如对于一个给定的ledger L, 所有存储了ledger L的fragment的bookie构成一个组，这个组就叫作`ensemble`。这个具体的关系可以参考我们上面这张图。

###### 开发动机

开发BookKeeper最初的动机来源于Hadoop生态。在HDFS中，有一类特殊的节点叫`NameNode`, 它以一种可靠的方式记录所有的操作，以确保在crash的情况下能够恢复。这个`NameNode`就是BookKeeper最初的灵感 来源。现在BookKeeper可以服务的应用已经扩展到所有需要`append-based`存储系统的应用上。BookKeeper为这些应用提供了下列优势：

* 高效的写入性能
* 通过复本实现高效的故障转移
* 高吞吐量

###### Metadata存储

BookKeeper目前使用ZooKeeper来存储`ledgers`相关信息和有效的`bookies`。

#### BookKeeper 协议

BookKeeper使用特定的复制协议来保证entries的持久化存储。复制log由ledgers的有序列组成。

##### Ledgers

Ledgers是BookKeeper的基础构建块，BookKeeper确保对它作持久化存储。

Ledgers由metadata和entries组成。其中metadata存储在Zookeeper上，它提供了CAS的原子操作。Entires存储在被称为bookies的存储节点上。

一个Ledger同时只能有一个写者，但可以有多个读者。

##### Ledger metadata

一个Ledger的metadata包含下列部分：

| Parameter         | Name   | Meaning                                                      |
| :---------------- | :----- | :----------------------------------------------------------- |
| Identifer         |        | 64位整型，在整个系统中是唯一的                               |
| Ensemble size     | **E**  | 这个Ledger需要被存储在几个bookie上                           |
| Write quorum size | **Qw** | 每条 entry需要被同步到的最大复本数                           |
| Ack quorum size   | **Qa** | 每条entry需要被同步到的最小复本数，即只有Qa个复本回了ack, 才会给客户端返回成功 |
| Current state     |        | 当前ledge的状态： `OPEN`, `CLOSED`, or `IN_RECOVERY`         |
| Last entry id     |        | 当前ledger的最后一条entry的id, 如果当前ledger状态不为 closed, 则为 null |

下面是从zk上获取到的metadata的信息：

```assembly
BookieMetadataFormatVersion     2
quorumSize: 2
ensembleSize: 3
length: 0
lastEntryId: -1
state: OPEN
segment {
  ensembleMember: "xxx.xxx.xxx.xxx:3181"
  ensembleMember: "xxx.xxx.xxx.xxx:3181"
  ensembleMember: "xxx.xxx.xxx.xxx:3181"
  firstEntryId: 0
}
digestType: HMAC
password: "some-password"
ackQuorumSize: 2

```

当创建一个ledger时，下列条件必须能够被满足：

E >= Q(w) >= Q(a)

即这个 ledger ensemble(E)必须大于等于write quorum size (**Qw**), 它又必须大于等于ack quorum size (**Qa**)。如果上述条件不满足，ledger将创建失败。

##### Ensembles

当一个Ledger被创建后，会选出E个bookies来存储当前的Ledger, 这E个bookies合起来称为当前ledger的Ensemble。同一个Ledger可以有多个ensembles，但是一条entry只能属于唯一的一个ensemble。当这个ledger有新的fragment产生时，它的ensemble将改变。

我们来看一个例子，在这个 ledger里，ensembles的大小是3, 它有两个fragments,因此也就有两个ensembles，一个从entry 0开始，一个从entry 12开始。这两个的ensemble的组成bookies是不同的。这可能发生在bookiea(B1)发生了故障，然后生成了新的fragment， 亲的fragment的ensemble里将不会再包括故障的B1, 用其他有效的节点来替换。

| First entry | Bookies    |
| :---------- | :--------- |
| 0           | B1, B2, B3 |
| 12          | B4, B2, B3 |

##### Write quorums

每条entry都会被写到Q(w)个节点，这被称为针对这条entry的write quorum。这个write quorum是长度是Q(w)的ensemble的子序列，并且从bookies[entryid % **E**]处的bookie开始，这样设计是为了平衡写入的负载。

比如有一个ledger, 它的**E** = 4, **Qw**, and **Qa** = 2， 且它的ensemble由B1, B2, B3和B4组成，则它的前6个entries的write quorums是：

| Entry ID | Write quorum |
| :------- | :----------- |
| 0        | B1, B2, B3   |
| 1        | B2, B3, B4   |
| 2        | B3, B4, B1   |
| 3        | B4, B1, B2   |
| 4        | B1, B2, B3   |
| 5        | B2, B3, B4   |

##### Ack quorums

这个ack quorums是长度为Q(a)的write quorums的子集。如果有Q(a)个bookies都ack了这条entry, 我们就说这条entry已经被完全复制了，不会丢失了。比如设置了一个ledger的复本数Q(w)是3, Q(a)是2, 那么写入时会同时向三个复本写入数据，其中两个写成功后就会给客户端回ack, 表时已经写入成功。

###### 协议保证

BookKeeper保证有Q(a) - 1个节点故障也不会有数据丢失。

因为针对一条entry, 只有当Q(a)个节点都写成功后，才会给客户端返回成功，如果坏了Q(a) - 1个节点，还可以从Q(a)里剩下的一个节点读出数据来。

BookKeeper确保：

1. 所有对ledger的更新都会以与它们被写入的相同的顺序被读出来;
2. 所有的客户端都会以相同的顺序读取到对ledger的更新。

#### 写数据到ledgers

写者确保entry id是单调递增的。一旦entry被持久化到磁盘，这个bookie就会ack这次写入操作。一旦write quorum中有Q(a)个bookies都ack了这个写入操作，那个这次的写入操作的ack将返回给客户端，并且此时比当前entry id小的所有entries都已经是ack回了客户端。

写入的entry包含ledger id, entry id, last add confirmed 和payload。这个last add confirmed是客户端收到的最新的ack对应的entry id。

另一客户端可以读取到ledger中的entry,  直到最后被确认的一条消息，我们已经确保了所有的entries都已经复制到Q(a)的节点， 因此所有将来的读者也能够读取到它们。但是，为了能像上述的方式读取，这个ledger需要以非fencing的方式打开，否则它将终止这个写入。

如果某个节点写入时失败，写者将创建一个新的ensemble，在这个新的ensemble里将不包含之前写入失败的节点。BookKeeper将在这个新的ensemble中创建新的fragment，并且entry是从每一条没有被ack的msg开始。创建新的fragment会使用一个CAS操作将metadata写入zk。如果当前有其他一些对zk的操作，这个CAS写可能会失败。这种并发更改的操作可能由recovery或rereplication导致, 此时BookKeeper会重新读取metadata，如果这个ledger的状态不再是`OPEN`，系统将发送一个error到客户端。

##### 写者关闭一个ledger

写者关闭一个ledger的过程是简单明了的。这个写者通过一个CAS的写操来更新metadata到zk，改变其状态为`CLOSED`并且设置这个last entry为我们ack到客户端的最后一条entry。

如果CAS写失败，它意味着有其他操作正在更改metadata。我们将重新读取metadata，并且如果其状态不是`CLOSED`，我们会继续尝试关闭这个ledger。如果状态是`IN_RECOVERY`, 会发送error到客户端。如果状态是`CLOSED`并且last entry和我们ack到客户端的是相同的，那我们成功完成了这次close操作。如果状态是`CLOSED`并且last entry和我们ack到客户端的是不相同的, 会发送error到客户端。



#### 前期需求

一个典型的BookKeeper集群有bookies集合和ZooKeeper构成。这个bookies的精确的数量取决于你选择的quorum模型，期望的吞量和并发访问的客户端的数量。

这个bookies的最小数量取决于集群的部署类型：

* 对于自校验的entries你需要至少运行三个bookies。在这种模式下，客户端将message和校验码一起存储在每个entry中。
* 针对 `generic`类的entries你需要至少四个bookies。

可以运行在单个集合中的bookies数没有上限

#### 性能

为了达到最优的性，BookKeeper要求每台server至少有两块磁盘，原因是BookKeeper在写入数据时会有三种数据需要落盘：journals, entry logs和infex files, 这里journals是实时每条落盘（最好是SSD），其余两类是batch的方式定时 flush到磁盘，这里说至少两块磁盘，是把上述的两类分不同磁盘存储。当然只使用一块磁盘也没有问题，只是性能上会有损失。

#### ZooKeeper

原文说对Zookeeper节点数据没有限制，也可以使用单节点的standalone模式，但出于高可用的考量，我们还是推荐集群方式的部署，一般是3个或5个节点。

#### 配置bookies

Bookies有很多的配置参数，位于配置文件`conf/bk_server.conf`下。我们下面只介绍几个最主要的配置参数

* bookiePort: bookie所监听的对外提供服务的tcp端口， 默认是3181
* metadataServiceUri：用来存储metadata的service uri, 通常使用zookeeper, 形如 `metadataServiceUri=zk+hierarchical://10.xxx.xxx.xxx:2181;10.xxx.xxx.xxx:2181;10.xxx.xxx.xxx:2181/ledgers`
* journalDirectory: 存储journal的目录
* ledgerDirectories:存储entry log的目录，它可以有多个，用逗号分隔
* indexDirectories: 存储index的目录

