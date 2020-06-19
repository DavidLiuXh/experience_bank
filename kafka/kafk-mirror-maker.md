- 说是图解,其实不是用正规的UML, 不是流程图, 不是时序图, 不是...
- Mirror Maker 逻辑简单, 代码不多, 就一个 sacala文件: core/src/scala/kafka/tools/MirrorMaker.scala文件,里面使用了producer和consumer的java客户端SDK
- Mirror Maker 这东西在使用中发现对同步的机房间网络质量要求还是比较高的, 特别是异地机房间
- 使用Mirror Maker作同步需要加上对consumer lag的监控, 如果lag一直增加, 源broker offset也一直在增加,那可能是网络有问题或者是mm进程hang住了, 不要犹豫,重启mm吧~
- 部署的时候尽可能部署在和目的集群相同的机房
- 不多说,上图!


![kafka-mirror-maker.jpeg](http://upload-images.jianshu.io/upload_images/2020390-388a7b5c4c92f8ed.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)