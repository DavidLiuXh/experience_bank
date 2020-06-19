* Kafka消费后都会提交保存当前的消费位置offset, 可以选择保存在zk, 本地文件或其他存储系统;
* Kafka 0.8以后提供了`Coordinator`的角色,.`Coordinator`除了可以来协调消费的`group`作balance外, 还接受 [OffsetCommit Request](http://kafka.apache.org/protocol.html#The_Messages_OffsetCommit), 用来存储消费的offset到Kafka本身中.具体可参考[Kafka的消息是如何被消费的?](http://www.jianshu.com/p/1aba6e226763);
---
###### Kafka 0.8以前的版本
- 绝大部分的offset应该都是写到zookeeper上, 类似`/consumers/[consumer group]/offsets/[topic]/[partition]`
- 如果不想重启消费进程就能reset, 可以在zk上创建一个新节点,专门用来记录需要reset的offset位軒,然后代码里watch这个节点, 获取到需要重置到的offset值,然后在发送[Fetch Request](http://kafka.apache.org/protocol.html#The_Messages_Fetch)时使用这个新的offset值即可;
###### Kafka 0.10以后的版本
- Kafka 引入了`Timestamp`, 具体可参考[Add a time based log index](https://cwiki.apache.org/confluence/display/KAFKA/KIP-33+-+Add+a+time+based+log+index), 这样就可以方便的根据时间来获取并回滚相应的消费啦,真是太方便了;
- 不仅如此, Kafka还提供了专门的工具来作Offset rest, 具体不累述,请参考[Add Reset Consumer Group Offsets tooling](https://cwiki.apache.org/confluence/display/KAFKA/KIP-122%3A+Add+Reset+Consumer+Group+Offsets+tooling)
###### Kafka 0.9.0.1版本
- 这个版本你当然还是可以将offset保存在zk中, 然后使用上面提到的方法重置;
- 我们现在重点来讨论下将offset保存到kafka系统本身的,其实就是存到一个内部的叫__consumer_offsets中,具体可参考[Kafka的消息是如何被消费的?](http://www.jianshu.com/p/1aba6e226763);
- Kafka提供自动reset的配置
   1. `auto.offset.reset`
        1.1  **smallest** : 自动重置到最小的offset, 这个最小的offset不一定是0, 因为msg可能会被过期删除掉;
        1.2 **largest** : 自动重置到最大的offset;
  2. 这个配置只有在**当前无法获取到有效的offset**时才生效;
      2.1 全新的group;
      2.2 已存在的group, 但很久没有提交过offset, 其保存在__consumer_offsets里的信息将被compact并最终清除掉;
- 需要手动reset时, 并没有像Kafka 0.10(11)版本那样提供给我们工具, 那怎么办? 只能自已搞, 下面提供一个思路:
   1. 确定需要重置到的offset:
       1.1 如果想重置到最新或最旧的offset, 可能通过kafka的命令行工具获取:
`kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list  [broker list] --topic [topic name] --time [-1:获取最新offset, -2:获取最旧offset]`
       1.2 也可以通过代码来获取, 可以使用[librdkafka](https://github.com/edenhill/librdkafka)的`rd_kafka_query_watermark_offsets`函数来获取;
   2. 重置offset, 以使用[librdkafka](https://github.com/edenhill/librdkafka)提供的接口为例:
      2.0 需要先停掉需重置的group的所有消费进程,可以使用`rd_kafka_list_groups`来获取当前消费 gropu的详情;
      2.1 使用`rd_kafka_topic_partition_list_set_offset`来设置需要重置的partiton的offset；
      2.2 调用`rd_kafka_subscribe`和`rd_kafka_consumer_poll`来等待group完成balance；
      2.3 调用`rd_kafka_commit`来完成重置的offset的提交;
   3. 当然[librdkafka](https://github.com/edenhill/librdkafka)和kafka api都提供了`seek`接口,也可以用来设置offset;
   4. 我们也开源了一个工具来作重置这件事：[KafkaOffsetTools](https://github.com/DavidLiuXh/KafkaOffsetTools)
- 如果不是想重置到最新或最旧的offset, 而是想重置到某一时间点的offset, 该怎么办?
  1. 这个版本不支持timestamp, 如果不想对kafka源码作改动的话, 可以定时获到group的消费offset, 然后写入到外部存储系统, 比如redis;
  2. 需要重置时,从外部存储系统根据时间点来获到到当时的offset, 由于是定时采样,不一定能完全匹配上指定的时间点,但可以取与其最接近的时间点.


# [Kafka源码分析-汇总](http://www.jianshu.com/p/aa274f8fe00f)