# 对nio的封装:Selector类
* 所在文件: clients/src/main/java/org/apache/kafka/commmon/network/Selector.java
* 源码中的注释: 
> A nioSelector interface for doing non-blocking multi-connection network I/O. This class works with NetworkSend} and NetworkReceive to transmit size-delimited network requests and  responses.
* 重要函数解析:
(1) **register(String id, SocketChannel socketChannel)**: 注册这个socketChannel到一个nio selector, 将其读事件添加到selector的监听队列; 这个socketChannel通常是服务器接收到的客户端的连接:
```
SelectionKey key = socketChannel.register(nioSelector, SelectionKey.OP_READ);
```
同时创建KafkaChannel, 负责实际的数据接收和发送:
```
KafkaChannel channel = channelBuilder.buildChannel(id, key, maxReceiveSize);
key.attach(channel);
this.channels.put(id, channel);
```
上面的id即为我们在上篇介绍的非常重要的ConnectionId;
(2) **connect**: 使用nio的SocketChannel连接到给定的地址,并且注册到nio selector,同时也创建了KafkaChannel,负责实际的数据接收和发送;
```
SocketChannel socketChannel = SocketChannel.open();
socketChannel.configureBlocking(false);
Socket socket = socketChannel.socket();
socket.setKeepAlive(true);
socketChannel.connect(address);
SelectionKey key = socketChannel.register(nioSelector, SelectionKey.OP_CONNECT);
KafkaChannel channel = channelBuilder.buildChannel(id, key, maxReceiveSize);
key.attach(channel);
this.channels.put(id, channel);
```
(3) **poll**:  核心函数:
>Do whatever I/O can be done on each connection without blocking. This includes completing connections, completing disconnections, initiating new sends, or making progress on in-progress sends or receives.

 ***处理作为客户端的主动连接事件:***
```
if (key.isConnectable()) {
        channel.finishConnect();
         this.connected.add(channel.id());
         this.sensors.connectionCreated.record();
}
```
***处理连接建立或接收后的ssl握手或sasl签权操作:***
```
if (channel.isConnected() && !channel.ready())
      channel.prepare();
```
***处理触发的读事件:***
```
if (channel.ready() && key.isReadable() && !hasStagedReceive(channel)) {
          NetworkReceive networkReceive;
          while ((networkReceive = channel.read()) != null)
                     addToStagedReceives(channel, networkReceive);
}
```
使用一个while循环力求每次读事件触发时都读尽可能多的数据;
channel.read()里会作拆包处理(后面会讲到),返回非null表示当前返回的NetworkReceive里包含了完整的应用层协议数据;
***处理触发的写事件:***
```
if (channel.ready() && key.isWritable()) {
         Send send = channel.write();
          if (send != null) {
                 this.completedSends.add(send);
                  this.sensors.recordBytesSent(channel.id(), send.size());
          }
}
```
需要发送数据通过调用Selector::send方法,设置封装了写数据的NetworkSend,再将这个NetworkSend通过KafkaChannel::setSend接口设置到KafkaChannel,同时将写事件添加到selector的监听队列中,等待写事件被触发后,通过KafkaChannel::write将数据发送出去;
***addToCompletedReceives()***
将当前接收到的完整的的request到添加到completedReceives中,上一篇中介绍的SocketServer会作completedReceives中取出这些request作处理;

# 封装对单个连接的读写操作:KafkaChannel类
* 所在文件: clients/src/main/java/org/apache/kafka/common/network/KafkaChannel.java
* 包括transportLayer和authenticator, 完成ssh握手,sasl签权,数据的接收和发送;

#  传输层:TransportLayer类
* 所在文件 clients/src/main/java/org/apache/kafka/common/network/TransportLayer.java
* 两个子类: PlanintextTransportLayer和SslTransportLayer
* PlanintextTransportLayer的实现主要是通过NetworkReceive和NetworkSend;
* SslTransportLayer的实现主要是通过SocketChannel,ByteBuffers和SSLEngine实际了加密数据的接收和发送(看到ssl就头大啊,这部分先忽略~~~);

# Kafka协议的包结构:
* 前4个字节固定, 值是后面的实际数据的长度;
* NetworkReceive: 接收时先接收4个字节, 获取到长度,然后再接收实际的数据;
* NetworkSend: 发送时实际数据前先加上4个字节的数据长度再发送;

# 上图:

![selector.png](http://upload-images.jianshu.io/upload_images/2020390-ec4d1136251beca6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 下一篇开始讲Kafka的服务端配置