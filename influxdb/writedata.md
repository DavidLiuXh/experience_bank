###### Influxdb数据写入流程
![write_flow.png](https://upload-images.jianshu.io/upload_images/2020390-90a06a06053405c3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 这个之前我之提供了一张图，其实Influxdb的写入还是比较简单的，当然这个简单是建立在你对它的存储引擎有了一定的了解之上的。我们可以通过之前的文章来回顾一下：
* [Influxdb中TSI索引解析之TSL文件](https://www.jianshu.com/p/158f30b7d040)
* [InfluxDB中的inmem内存索引结构解析](https://www.jianshu.com/p/ea7d0cb37405)
* [Influxdb中TSM文件结构解析之WAL](https://www.jianshu.com/p/81b8254fdb0d)
* [Influxdb中TSM文件结构解析之读写TSM](https://www.jianshu.com/p/f6be57926b84)
* [Influxdb中的Compaction操作](https://www.jianshu.com/p/10379e77920e)
* [Influxdb存储引擎Engine启动流程](https://www.jianshu.com/p/ca30f62adb1a)

###### Influxdb在存储层使用了双引擎：一个用来存储索引，一个用来存储数据，因此很自然地数据写入时需要同时写入这两个引擎，虽然听起来增加了复杂度，但代码层级其实分隔得还算工整
###### 写入的数据以`models.Points`的形式传递到引擎层，下面这张图是源码中`Store`所设计的组件的结构图
![store_arc.png](https://upload-images.jianshu.io/upload_images/2020390-3b367cd75d24451b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
