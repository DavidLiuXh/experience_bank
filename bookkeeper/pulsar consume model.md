| topic 方式 | sub model | Java Sdk Action                                              |
| ---------- | --------- | ------------------------------------------------------------ |
| Partitions | Exclusive | 只能 client.newConsumer()一个Consumer, 实现上其内部针对每一个parition（即子topic）分别Subscribe， 即实际上是创建了三台子consumer；最终消费的Msg顺序不保证; |
| Partitions | Failover  | 可以启动任意多个 consumer, 但同时只有和partition数目相同的consumer  可以接收到数据；并且后启动的 consumer会踢掉先启动的consumer来接收到数据； |
|            |           |                                                              |
|            |           |                                                              |
|            |           |                                                              |

