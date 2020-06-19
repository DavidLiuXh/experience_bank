* 在享用了这么久kafka提供的各种福利后, 咱们也来精进一下,看看kafka的各部分实现,知其然也知一点所以然;
* 题目起得有点大,其实完全是临时起意,希望能坚持下去;
* 本身其实不是java程序员,scala也是半吊子, 但是特别喜欢scala啊~~~
* Kafka最近的版本更新有点快, 但是这一系列文章是基于kafka 0.9.1版本;
* 这里的文章不会事无巨细,但求将主脉络理清.
***
# Kafka的网络层模型概述
* 这个模型其实一点也不神秘,很质朴,很清晰,也很好用,引用源码中的一句话:
>The threading model is 1 Acceptor thread that handles new connections Acceptor has N Processor threads that each have their own selector and read requests from socketsM Handler threads that handle requests and produce responses back to the processor threads for writing
* 再来张图:
![网络模型.png](http://upload-images.jianshu.io/upload_images/2020390-267f4c72690e1d1c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* Acceptor 作两件事: 创建一堆worker线程；接受新连接, 将新的socket指派给某个 worker线程;
* Worker线程处理若干个socket,接受请求转给各种handler处理,response再经由worker线程发送回去.
* 总结起来就是个[半同步半异步模型](https://github.com/DavidLiuXh/lightningserver).
# Kafka的网络层模型实现
* 虽然kafka用scala实现,但里面也用了大量的java类, 这部分主要是用了[NIO](http://tutorials.jenkov.com/java-nio/index.html)；
* 主要实现文件:core/src/main/scal/kafka/network/SocketServer.scala,里面包括了SocketServer, Acceptor, Processor等;
* 数据传输层实现:clients/src/main/java/org/apache/kafka/common/network,里面包括了Channel,TransportLayer,Authenticator等.
* 下一篇咱们开始进入到具体的实现...

#  [Kafka源码分析-网络层-2](http://www.jianshu.com/p/713df18cf47c)

###### [Kafka源码分析-汇总](http://www.jianshu.com/p/aa274f8fe00f)