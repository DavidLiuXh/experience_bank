### 导引

![qcm.png](https://upload-images.jianshu.io/upload_images/2020390-bad25e20599b883c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

[KafkaBridge](https://github.com/Qihoo360/kafkabridge) 封装了对Kafka集群的读写操作，接口极少，简单易用，稳定可靠，支持c++/c、php、python、golang等多种语言，并特别针对php-fpm场景中作了长连接复用的优化，已在360公司内部广泛使用。
   
   ### 前言
   * 众所周知，Kafka是近几年来大数据领域最流行的分布式流处理平台。它最初由LinkedIn公司开发， 已于2010年贡献给了Apache基金会并成为顶级开源项目, 本质上是一种低延时的、可扩展的、设计内在就是分布式的，分区的和可复制的消息系统;
   * Kafka在360公司内部也有相当广泛的使用，业务覆盖搜索，商业广告，IOT, 视频，安全， 游戏等几乎所有核心业务，每天的写入流量近1.2PB, 读取流量近2.4PB;
   * Kafka官方提供了Java版本的客户端SDK, 但因360公司内部产品线众多，语言几乎囊括目前所有主流语言，所以我们研发了Kafka客户端SDK ——　KafkaBridge;

   ### 简介
   * KafkaBridge 底层基于 [librdkafka](https://github.com/edenhill/librdkafka), 与之相比封装了大量的使用细节，简单易用，使用者无需了解过多的Kafka系统细节，只需调用极少量的接口，就可完成消息的生产和消费;
   * 针对使用者比较关心的消息生产的可靠性，作了近一步的提升；
   * 开源地址：[https://github.com/Qihoo360/kafkabridge](https://github.com/Qihoo360/kafkabridge)

   ### 特点
   * 支持多种语言：c++/c、php、python、golang, 且各语言接口完全统一;
   * 接口少，简单易用;
   * 针对高级用户，支持通过配置文件调整所有的librdkafka的配置;
   * 在非按key写入数据的情况下，尽最大努力将消息成功写入;
   * 支持同步和异步两种数据写入方式;
   * 在消费时，除默认自动提交offset外，允许用户通过配置手动提交offset;
   * 在php-fpm场景中，复用长连接生产消息，避免频繁创建断开连接的开销;

   ### 编译
   * 编译依赖于 [librdkafka](https://github.com/edenhill/librdkafka), [liblog4cplus](https://sourceforge.net/projects/log4cplus/), [boost(仅依赖于若干个头文件)](https://www.boost.org/);
   * 对于C++/C使用 [CMake](https://cmake.org/) 编译;
   * 对于Python, Php, Golang使用 [swig](http://www.swig.org/) 编译;
   * 每种语言都提供了自动编译脚本，方便使用者自行编译。

   ## 使用
   #### 数据写入
   * 在非按key写入的情况下，sdk尽最大努力提交每一条消息，只要Kafka集群存有一台broker正常，就会重试发送;
   * 每次写入数据只需要调用*produce*接口，在异步发送的场景下，通过返回值可以判断发送队列是否填满，发送队列可通过配置文件调整;
   * 在同步发送的场景中，*produce*接口返回当前消息是否写入成功，但是写入性能会有所下降，CPU使用率会有所上升,推荐还是使用异步写入方式;
   * 我们来简单看一下写入kafka所涉及到的所有接口:
   ~~~
   //初始化接口
   bool QbusProducer::init(const string& broker_list, const string& log_path, const string& config_path, const string& topic)
   //写入数据接口
   bool QbusProducer::produce(const char* data, size_t data_len, const std::string& key)
   //不再需要写入数据时，需要调用的清理接口，必须调用 
   void QbusProducer::uninit()
   ~~~

   * 具体使用可以参考源码中的实例;

   #### 数据消费
   * 消费只需调用subscribeOne订阅topic（也支持同时订阅多个topic），然后执行start就开始消费，当前进程非阻塞，每条消息通过callback接口回调给使用者;
   * sdk还支持用户手动提交offset方式，用户可以通过callback中返回的消息体，在代码其他逻辑中进行提交。
   * 下面是消费接口，以c++为例：
   ~~~
   //初始化接口
   bool QbusConsumer::init(const string& string broker_list, const string& string log_path, const string& string config_path, QbusConsumerCallback& callback)
   //订阅需要消费的消息
   bool QbusConsumer::subscribeOne(const string& string group, const string& string topic)
   //开始消费
   bool QbusConsumer::start()
   //停止消费
   void QbusConsumer::stop()
   ~~~

   ## 性能测试
   * kafka 集群三台broker, 除测试用topic外，无其他topic的读写操作;
   * 测试用topic有3个partition;
   * Producer单实例，单线程;
   * Topic无复本下测试：
     1. 单条消息 100 byte, 发送 1百万 条消息，耗时 1.7 秒;
       2. 单条消息 1024 byte, 发送 1百万 条消息，耗时 13 秒;
       * Topic有2复本下测试：
         1. 单条消息 100 byte, 发送 1百万 条消息，耗时 1.7 秒;
           2. 单条消息 1024 byte, 发送 1百万 条消息，耗时 14 秒;  

           ## 写在最后
           * KafkaBridge 一直在360公司内部使用，现在已经开源，有疏漏之处，欢迎广大使用者批评指正，也欢迎更多的使用者加入到 KafkaBridge 的持续改进中。
           * 开源地址：[KafkaBridge](https://github.com/Qihoo360/kafkabridge)
           * QQ群：![截图20181010174725271.jpg](https://upload-images.jianshu.io/upload_images/2020390-9ba827f2b1fb89d0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
