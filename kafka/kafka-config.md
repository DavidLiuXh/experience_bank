* 几乎没有服务不需要配置:命令行参数 or 配置文件；
* 配置项多了, 设置起来太麻烦；配置项少, 不够灵活；
***
# Kafka的配置
* Kafka的配置相当丰富,写成本手册一点问题都没有, [官网配置说明](http://kafka.apache.org/090/documentation.html#configuration);
* 与Kafka Server相关的配置分类:
Zookeeper
General
Authorizer
Socket Server
Log
Replication
Controlled shutdown
Offset management
Quota
Kafka Metrics
SSL
Sasl

# Kafka配置设置实现
* 所在文件: core/src/main/scala/kafka/server/KafkaConfig.scala;
* **object Defaults:** 定义了所有的配置项默认值;
* **object KafkaConfig:** 定义了所有的配置项名称:
```
PrincipalBuilderClassProp = SslConfigs.PRINCIPAL_BUILDER_CLASS_CONFIG
...
```
说明文档:
```
/* Documentation */
  /** ********* Zookeeper Configuration ***********/
  val ZkConnectDoc = "Zookeeper host string"
...
```
创建了configDef, 是一个ConfigDef类对象:
```
private val configDef = {
    import ConfigDef.Importance._
    import ConfigDef.Range._
    import ConfigDef.Type._
    import ConfigDef.ValidString._

    new ConfigDef().define(...).define(...)...
```
作为Class KafkaConfig的[伴生类](),定义了创建KafkaConfig对象的工厂方法:
```
def apply(props: java.util.Map[_, _]): KafkaConfig = new KafkaConfig(props, true)
```

# 通用Config类:AbstractConfig
* 所在文件: clients/src/main/java/org/apache/kafka/common/config/AbstractConfig.java
* 源码中注释:
>A convenient base class for configurations to extend.
This class holds both the original configuration that was provided as well as the parsed

* 构造函数:
```
public AbstractConfig(ConfigDef definition, Map<?, ?> originals, Boolean doLog)
```
originals表示所有被用户设置了的参数;
definition表示所有的配置项,包默认值;
通过调用`definition.parse(this.originals)`得到使用用户设置参数更新后的所有配置项和值;
* 提供一系列的get方法,返回相应配置的值:
getInt
getShort
getLong
getDouble
getList
getBoolean
getString
getPassword
getClass

# 通用config构建类:ConfigDef
* 所在文件: clients/src/main/java/org/apache/kafka/common/config/ConfigDef.java
* **define:** 加入某一配置项,包括其name, type, defaultValue, importance, documentation等;
* **parse(Map<?, ?> props):** 使用props来更新一组配置项;
