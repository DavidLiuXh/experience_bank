* 我们一般都是使用`bin/kafka-server-start.sh`脚本来启动;
* 从`bin/kafka-server-start.sh`可以知道此脚本用法:
`echo "USAGE: $0 [-daemon] server.properties [--override property=value]*"`
(1) `server.properties`为配置文件路径, 这里`config/server.properties`有一个配置文件的模板,里面就是一行行的`key=value`;
(2) `--override property=value`是若干个可项的参数, 用来覆盖server.properties配置文件中同名的配置项;
* 从`bin/kafka-server-start.sh` 最后一行`exec $base_dir/kafka-run-class.sh $EXTRA_ARGS kafka.Kafka "$@"`可知, Kafka启动时的入口类为`kafka.Kafka`, 我们直接来看这个类;
***
# Kafka启动入口类:kafk.Kafak
* 所在文件: core/src/main/scala/kafka/Kafka.scala
* 定义: `object Kafka extends Logging`
* main函数:
```
      val serverProps = getPropsFromArgs(args)
      val kafkaServerStartable = KafkaServerStartable.fromProps(serverProps)

      // attach shutdown handler to catch control-c
      Runtime.getRuntime().addShutdownHook(new Thread() {
        override def run() = {
          kafkaServerStartable.shutdown //捕获control-c中断,停止当前服务
        }
      })

      kafkaServerStartable.startup //启动服务
      kafkaServerStartable.awaitShutdown //等待服务结束
```
使用getPropsFromArgs方法来获取各配置项, 然后将启动和停止动作全部代理给`KafkaServerStartable`类;

# Kafka启动代理类:KafkaServerStartable

* 伴生对象: `object KafkaServerStartable` 提供fromProps方法来创建 `KafkaServerStartable`;
* KafkaServerStartable对象创建时会同时创建 `KafkaServer`, 这才是真正的主角;
```
def startup() {
    try {
      server.startup()
    }
    catch {
      case e: Throwable =>
        fatal("Fatal error during KafkaServerStartable startup. Prepare to shutdown", e)
        // KafkaServer already calls shutdown() internally, so this is purely for logging & the exit code
        System.exit(1)
    }
  }

  def shutdown() {
    try {
      server.shutdown()
    }
    catch {
      case e: Throwable =>
        fatal("Fatal error during KafkaServerStable shutdown. Prepare to halt", e)
        // Calling exit() can lead to deadlock as exit() can be called multiple times. Force exit.
        Runtime.getRuntime.halt(1)
    }
  }

  /**
   * Allow setting broker state from the startable.
   * This is needed when a custom kafka server startable want to emit new states that it introduces.
   */
  def setServerState(newState: Byte) {
    server.brokerState.newState(newState)
  }

  def awaitShutdown() = 
    server.awaitShutdown
```