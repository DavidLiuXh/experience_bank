* 先了解Reqeust和Response的构成, 有助于我们分析各种请求的处理过程;
* Kafka的Request基本上分为client->server和server->server两大类;
***
# 基础数据结构类:
## Type类:
* 所在文件: clients/src/main/java/org/apache/kafka/common/protocol/types/Type.java
* 这是一个`abstrace class`, 主要是定义了ByteBuffer与各种Object之间的序列化和反序列化;
```
public abstract void write(ByteBuffer buffer, Object o);
public abstract Object read(ByteBuffer buffer);
public abstract Object validate(Object o);
public abstract int sizeOf(Object o);
public boolean isNullable();
```
* 定义了若干Type类的实现类:
```
public static final Type INT8
public static final Type INT16
public static final Type INT32
public static final Type INT64
public static final Type STRING
public static final Type BYTES
public static final Type NULLABLE_BYTES
```

## ArrayOf类:
* 所在文件:  clients/src/main/java/org/apache/kafka/common/protocol/types/ArrayOf.java
* Type类的具体实现, 是Type对象的数组类型;
 
## Field类:
* 所在文件: clients/src/main/java/org/apache/kafka/common/protocol/types/Field.java
* 定义了在这个schema中的一个字段;
* 成员:
```
    final int index;
    public final String name;
    public final Type type;
    public final Object defaultValue;
    public final String doc;
    final Schema schema;
```

## Schema类:
* 所在文件:  clients/src/main/java/org/apache/kafka/common/protocol/types/schema.java
* Schema类本身实现了`Type类`, 又包含了一个`Field类`对象的数组, 构成了记录的Schema;

## Sturct类:
* 所在文件: clients/src/main/java/org/apache/kafka/common/protocol/types/struct.java
* 包括了一个`Schema`对象; 一个`Object[] values`数组,用于存放Schema描述的所有Field对应的值;
```
    private final Schema schema;
    private final Object[] values;
```
* 定义了一系列`getXXX`方法, 用来获取schema中某个Field对应的值;
* 定义了`set`方法, 用来设置schema中某个Field对应的值;
* `writeTo` 用来将Stuct对象序列华到ByteBuffer;
* `Schema`就是模板,`Struct`负责特化这个模板,向模板里添数据,构造出具体的request对象, 并可以将这个对象与ByteBuffer互相转化;

# 协议相关类型:
## Protocol类: 
* 所在文件: clients/src/main/java/org/apache/kafka/common/protocol/Protocol.java
* 定义了各种Schema:
```
public static final Schema REQUEST_HEADER = new Schema(new Field("api_key", INT16, "The id of the request type."),
                                                           new Field("api_version", INT16, "The version of the API."),
                                                           new Field("correlation_id",
                                                                     INT32,
                                                                     "A user-supplied integer value that will be passed back with the response"),
                                                           new Field("client_id",
                                                                     STRING,
                                                                     "A user specified identifier for the client making the request."));
...
```

## ApiKeys类:
* 所在文件: clients/src/main/java/org/apache/kafka/common/protocol/ApiKeys.java
* 定义了所有Kafka Api 的ID和名字
* 如下:
```
    PRODUCE(0, "Produce"),
    FETCH(1, "Fetch"),
    LIST_OFFSETS(2, "Offsets"),
    METADATA(3, "Metadata"),
    LEADER_AND_ISR(4, "LeaderAndIsr"),
    STOP_REPLICA(5, "StopReplica"),
    UPDATE_METADATA_KEY(6, "UpdateMetadata"),
    CONTROLLED_SHUTDOWN_KEY(7, "ControlledShutdown"),
    OFFSET_COMMIT(8, "OffsetCommit"),
    OFFSET_FETCH(9, "OffsetFetch"),
    GROUP_COORDINATOR(10, "GroupCoordinator"),
    JOIN_GROUP(11, "JoinGroup"),
    HEARTBEAT(12, "Heartbeat"),
    LEAVE_GROUP(13, "LeaveGroup"),
    SYNC_GROUP(14, "SyncGroup"),
    DESCRIBE_GROUPS(15, "DescribeGroups"),
    LIST_GROUPS(16, "ListGroups");
```

# Request和Response相关类型
***每个Request和Response都由RequestHeader(ResponseHeader) + 具体的消费体构成;***
## AbstractRequestResponse类:
* 所在文件: clients/src/main/java/org/apache/kafka/common/request/AbstractRequestResponse.java
* 所有Request和Response的抽象基类
* 主要数据成员: `protected final Struct struct`
* 主要接口:
```
public int sizeOf()
public void writeTo(ByteBuffer buffer)
public String toString()
public int hashCode()
public boolean equals(Object obj)
```

## AbstractRequest类:
* 所在文件: clients/src/main/java/org/apache/kafka/common/request/AbstractRequest.java
* 继承自`AbstractReqeustResponse`类, 增加了接口:
`public abstract AbstractRequestResponse getErrorResponse(int versionId, Throwable e)`
* 最重要的是它提供了一个工厂方法用于从ByteBuffer来产生不同类型的具体的Request;
```
public static AbstractRequest getRequest(int requestId, int versionId, ByteBuffer buffer) {
        switch (ApiKeys.forId(requestId)) {
            case PRODUCE:
                return ProduceRequest.parse(buffer, versionId);
            case FETCH:
                return FetchRequest.parse(buffer, versionId);
            case LIST_OFFSETS:
                return ListOffsetRequest.parse(buffer, versionId);
            case METADATA:
                return MetadataRequest.parse(buffer, versionId);
            case OFFSET_COMMIT:
                return OffsetCommitRequest.parse(buffer, versionId);
            case OFFSET_FETCH:
                return OffsetFetchRequest.parse(buffer, versionId);
            case GROUP_COORDINATOR:
                return GroupCoordinatorRequest.parse(buffer, versionId);
            case JOIN_GROUP:
                return JoinGroupRequest.parse(buffer, versionId);
            case HEARTBEAT:
                return HeartbeatRequest.parse(buffer, versionId);
            case LEAVE_GROUP:
                return LeaveGroupRequest.parse(buffer, versionId);
            case SYNC_GROUP:
                return SyncGroupRequest.parse(buffer, versionId);
            case STOP_REPLICA:
                return StopReplicaRequest.parse(buffer, versionId);
            case CONTROLLED_SHUTDOWN_KEY:
                return ControlledShutdownRequest.parse(buffer, versionId);
            case UPDATE_METADATA_KEY:
                return UpdateMetadataRequest.parse(buffer, versionId);
            case LEADER_AND_ISR:
                return LeaderAndIsrRequest.parse(buffer, versionId);
            case DESCRIBE_GROUPS:
                return DescribeGroupsRequest.parse(buffer, versionId);
            case LIST_GROUPS:
                return ListGroupsRequest.parse(buffer, versionId);
            default:
                return null;
        }
    }
```
实现上是调用各个具体Request对象的`parse`方法根据bytebuffer和versionid来产生具体的Request对象;

## ProduceRequest类:
* 我们找其中一个ProduceRqeust类来分析一下, 这个类是客户端提交消息到broker时使用的请求;
* 所在文件: clients/src/main/java/org/apache/kafka/common/request/ProduceRequest.java
* 一个ProduceRequest包括下列字段:
```
    private final short acks;
    private final int timeout;
    private final Map<TopicPartition, ByteBuffer> partitionRecords;
```
* 构造函数`public ProduceRequest(Struct struct)`, 利用Struct里定义的Schame来从ByteBuffer反序列化出ProduceRequest对象;
```
public ProduceRequest(Struct struct) {
        super(struct);
        partitionRecords = new HashMap<TopicPartition, ByteBuffer>();
        for (Object topicDataObj : struct.getArray(TOPIC_DATA_KEY_NAME)) {
            Struct topicData = (Struct) topicDataObj;
            String topic = topicData.getString(TOPIC_KEY_NAME);
            for (Object partitionResponseObj : topicData.getArray(PARTITION_DATA_KEY_NAME)) {
                Struct partitionResponse = (Struct) partitionResponseObj;
                int partition = partitionResponse.getInt(PARTITION_KEY_NAME);
                ByteBuffer records = partitionResponse.getBytes(RECORD_SET_KEY_NAME);
                partitionRecords.put(new TopicPartition(topic, partition), records);
            }
        }
        acks = struct.getShort(ACKS_KEY_NAME);
        timeout = struct.getInt(TIMEOUT_KEY_NAME);
    }
```

## RequestHeader类:
* 所在文件: clients/src/main/java/org/apache/kafka/common/request/RequestHeader.java
* Request的消息头
* 主要成员:
```
    private static final Field API_KEY_FIELD = REQUEST_HEADER.get("api_key");
    private static final Field API_VERSION_FIELD = REQUEST_HEADER.get("api_version");
    private static final Field CLIENT_ID_FIELD = REQUEST_HEADER.get("client_id");
    private static final Field CORRELATION_ID_FIELD = REQUEST_HEADER.get("correlation_id");
```

# 关系图:

![request_response.png](http://upload-images.jianshu.io/upload_images/2020390-a6169a56f45b44eb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 实际上在 core/src/main/scala/kafka/api下也定义了各种Request和Response:
* 代码中的注释:
>NOTE: this map only includes the server-side request/response handlers. Newer
  request types should only use the client-side versions which are parsed with
  o.a.k.common.requests.AbstractRequest.getRequest()
```
     val keyToNameAndDeserializerMap: Map[Short, (String, (ByteBuffer) =>       
     RequestOrResponse)]=
     Map(ProduceKey -> ("Produce", ProducerRequest.readFrom),
        FetchKey -> ("Fetch", FetchRequest.readFrom),
        OffsetsKey -> ("Offsets", OffsetRequest.readFrom),
        MetadataKey -> ("Metadata", TopicMetadataRequest.readFrom),
        LeaderAndIsrKey -> ("LeaderAndIsr", LeaderAndIsrRequest.readFrom),
        StopReplicaKey -> ("StopReplica", StopReplicaRequest.readFrom),
        UpdateMetadataKey -> ("UpdateMetadata", UpdateMetadataRequest.readFrom),
        ControlledShutdownKey -> ("ControlledShutdown",   
        ControlledShutdownRequest.readFrom),
        OffsetCommitKey -> ("OffsetCommit", OffsetCommitRequest.readFrom),
        OffsetFetchKey -> ("OffsetFetch", OffsetFetchRequest.readFrom)
```
* 这部分作解析, 没有采用schema的形式, 是采用的直接读取方式:
```
def readFrom(buffer: ByteBuffer): ProducerRequest = {
    val versionId: Short = buffer.getShort
    val correlationId: Int = buffer.getInt
    val clientId: String = readShortString(buffer)
    val requiredAcks: Short = buffer.getShort
    val ackTimeoutMs: Int = buffer.getInt
    //build the topic structure
    val topicCount = buffer.getInt
    val partitionDataPairs = (1 to topicCount).flatMap(_ => {
      // process topic
      val topic = readShortString(buffer)
      val partitionCount = buffer.getInt
      (1 to partitionCount).map(_ => {
        val partition = buffer.getInt
        val messageSetSize = buffer.getInt
        val messageSetBuffer = new Array[Byte](messageSetSize)
        buffer.get(messageSetBuffer,0,messageSetSize)
        (TopicAndPartition(topic, partition), new ByteBufferMessageSet(ByteBuffer.wrap(messageSetBuffer)))
      })
    })
```

# 请求生成与保存
* 所有进来的请求最终会转换成 `RequestChannel::Request`, 保存在`RequestChannel`的`ArrayBlockingQueue[RequestChannel.Request]`中, 这个[前面章节](http://www.jianshu.com/p/713df18cf47c)已经讲过;

# [Kafka协议官网地址](http://kafka.apache.org/protocol.html#protocol_common)