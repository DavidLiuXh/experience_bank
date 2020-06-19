***
# Kafka网络层一哥:SocketServer类
* 所在文件: core/src/main/scala/kafka/network/SocketServer.scala;
* 统筹组织所有的网络层组件;
* **startup()方法**: 
(1) 根据配置的若干endpoint创建相应的Acceptor及相关联的一组Processor线程;
```
for (i <- processorBeginIndex until processorEndIndex) {
          processors(i) = new Processor(i,
            time,
            maxRequestSize,
            requestChannel,
            connectionQuotas,
            connectionsMaxIdleMs,
            protocol,
            config.values,
            metrics
          )
        }
val acceptor = new Acceptor(endpoint, sendBufferSize, recvBufferSize, brokerId,
processors.slice(processorBeginIndex, processorEndIndex), connectionQuotas)
```
(2) 创建Acceptor运行的线程并启动;
```
Utils.newThread("kafka-socket-acceptor-%s-%d".format(protocol.toString, 
                          endpoint.port), acceptor, false).start()
acceptor.awaitStartup()
```

# Kafka网络层头号马仔:Acceptor类
* 所在文件: core/src/main/scala/kafka/network/SocketServer.scala
* **Acceptor类对象创建**:
(1) 创建监听ServerSocket:
```
val serverChannel = ServerSocketChannel.open()
    serverChannel.configureBlocking(false)
    serverChannel.socket().setReceiveBufferSize(recvBufferSize)
    try {
      serverChannel.socket.bind(socketAddress)
      info("Awaiting socket connections on %s:%d.".format(socketAddress.getHostName, serverChannel.socket.getLocalPort))
    } catch {
      case e: SocketException =>
        throw new KafkaException("Socket server failed to bind to %s:%d: %s.".format(socketAddress.getHostName, port, e.getMessage), e)
    }
```
(2)  开启分配到的若干Processor:
```
this.synchronized {
    processors.foreach { processor =>
      Utils.newThread("kafka-network-thread-%d-%s-%d".format(brokerId, endPoint.protocolType.toString, processor.id), processor, false).start()
    }
  }
```
(3) **run()-**利用NIO的**selector**来接收网络连接:
```
var currentProcessor = 0
      while (isRunning) {
        try {
          val ready = nioSelector.select(500)
          if (ready > 0) {
            val keys = nioSelector.selectedKeys()
            val iter = keys.iterator()
            while (iter.hasNext && isRunning) {
              try {
                val key = iter.next
                iter.remove()
                if (key.isAcceptable)
                  accept(key, processors(currentProcessor))
                else
                  throw new IllegalStateException("Unrecognized key state for acceptor thread.")

                // round robin to the next processor thread
                currentProcessor = (currentProcessor + 1) % processors.length
              } catch {
                case e: Throwable => error("Error while accepting connection", e)
              }
            }
          }
        }
```
这里面最主要的就是`accept(key, processors(currentProcessor))`
(4) **[accept](#Processor_accept)**: 设置新连接socket的参数后交由Processor处理:
```
socketChannel.configureBlocking(false)
socketChannel.socket().setTcpNoDelay(true)
socketChannel.socket().setKeepAlive(true)
socketChannel.socket().setSendBufferSize(sendBufferSize)
processor.accept(socketChannel)
```

# Kafka网络层堂口扛把子:Processor类
* 所在文件:core/src/main/scala/kafka/network/SocketServer.scala;
* 从单个连接进来的request都由它处理;
* 每个Processor对象会创建自己的**nio selector**;
* 每个连接有唯一标识ConnectionId:`$localHost:$localPort-$remoteHost:$remotePort`,这个非常重要!!!
* **accept(socketChannel::SocketChannel)**:将新连接的SocketChannel保存到并发队列Q1中;
* **run()是核心**, 包裹在一个循环里,直接线程退出,`while(isRunning) {}`,里面依次调用如下函数:
(1) **configureNewConnections()**:从并发队列Q1里取出SocketChannel,添加到自身的nio selector中,监听读事件;
(2) **processNewResponses()**:处理当前所有处理完成的request相应的response, 这些response都是从RequestChannel获得(`requestChannel.receiveResponse`),根据request的类型来决定从当前连接的nio selector中暂时删除读事件监听/添加写事件/关闭当前连接;
(3) **selector.poll(300)**: 这个就不用解释了, 这个selector是对nio selector的又一封装,我们后面一章会讲到,它完成具体的数据接收和发送;
(4) `selector.completedReceives.asScala.foreach`: 处理当前所有的从selector返回的完整request,将其put到RequestChannel的一个阻塞队列里,供应用层获取并处理;同时会暂时删除些连接上的读事件监听:`selector.mute(receive.source)`;
(5) `selector.completedSends.asScala.foreach`: 处理当前所有的从selector返回的写操作,重新将读事件添加到连接的selector监听中`selector.unmute(send.destination)`;
(6) `selector.disconnected.asScala.foreach`: 处理当前所有将关闭的连接;

# Kafka网络层中间人:RequestChannel类
* 所在文件: core/src/main/scala/kafka/network/RequestChannel.scala;
* 保存所有从网络层拿到的完整request和需要发送的response;
* 一般是RequestHandler会周期性从RequestChannel获取request,并将response保存回RequestChannel;
* processNewResponses() 处理RequestChannel中所有的response;

# Kafka网络层看门小弟:ConnectionQuotas类
* 所在文件:core/src/main/scala/kafka/network/SocketServer.scala;
* 当某个IP到kafka的连接数过多时,将抛出`TooManyConnectionsException`异常;
* 实现就是通过加锁的加减计数;

# SocketServer 图解:

![SocketServer.png](http://upload-images.jianshu.io/upload_images/2020390-d2d99ee1c8253087.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 下一篇咱们来讲在Processor中使用的nio selector的又一封装,负责具体数据的接收和发送;