* kafka tcp协议格式： 头4个字节表示协议具体内容的长度;后面紧跟着是具体协议的内容; 在tcp流中这样的格式拆包非常简单明了;
* 具体协议部分，分为协议头和内容两部分, 具体的协议我在之前的[kafka 源码分析](https://www.jianshu.com/p/aa274f8fe00f)系列文章的[Kafka的Request和Response](https://www.jianshu.com/p/581169dec549)中有介绍;
* [Kafka官网的协议介绍](http://kafka.apache.org/protocol.html#protocol_common);
* Librdkafka对kafka协议作了c语言的封装, 分为Request和Response两种类型。
---
###### Request header
* 所在文件：src/rdkafka_proto.h
* [Kafka官网的说明](http://kafka.apache.org/protocol.html#protocol_common) 
* 定义：
```
struct rd_kafkap_reqhdr {
        int32_t  Size;
        int16_t  ApiKey;
        int16_t  ApiVersion;
        int32_t  CorrId;
        /* ClientId follows */
};
```
 需要注意的是：
  1. `int32_t Size`: 将request请求的总大小放到了这个request header的结构体中;
  2. 其他字段说明, 引用官网说明：
> **api_key** : The id of the request type.
**api_version** : The version of the API.
**correlation_id** : A user-supplied integer value that will be passed back with the response; 客户端产生的请求id, 在response中被回传回来，客户端用其来区分是针对哪个request的response;
**client_id** : A user specified identifier for the client making the request.
  3. 与官网不同的是，没有在此结构体中没有定义 `client_id`字段，我们在后面会讲到;
###### Response header
* 所在文件：src/rdkafka_proto.h
* [Kafka官网的说明](http://kafka.apache.org/protocol.html#protocol_common) 
* 定义：
```
struct rd_kafkap_reshdr {
	int32_t  Size;
	int32_t  CorrId;
};
```
 需要注意的是：
  1. int32_t Size: 将response的总大小放到了这个response header的结构kh体;
  2. `CorrId`与request中的correlation_id是一一对应关系;
###### Message消息格式:
* 所在文件: src/rdkafka_proto.h
* 目前官方有Msg Version v0, v1, v2三种格式, 具体可参考:[# [A Guide To The Kafka Protocol](https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol)
](https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol#AGuideToTheKafkaProtocol-Messagesets)
* Librdkafka这里对其各字段作了定义:
```
/**
 * MsgVersion v0..v1
 */
/* Offset + MessageSize */
#define RD_KAFKAP_MESSAGESET_V0_HDR_SIZE (8+4)
/* CRC + Magic + Attr + KeyLen + ValueLen */
#define RD_KAFKAP_MESSAGE_V0_HDR_SIZE    (4+1+1+4+4)
/* CRC + Magic + Attr + Timestamp + KeyLen + ValueLen */
#define RD_KAFKAP_MESSAGE_V1_HDR_SIZE    (4+1+1+8+4+4)
/* Maximum per-message overhead */
#define RD_KAFKAP_MESSAGE_V0_OVERHEAD                                   \
        (RD_KAFKAP_MESSAGESET_V0_HDR_SIZE + RD_KAFKAP_MESSAGE_V0_HDR_SIZE)
#define RD_KAFKAP_MESSAGE_V1_OVERHEAD                                   \
        (RD_KAFKAP_MESSAGESET_V0_HDR_SIZE + RD_KAFKAP_MESSAGE_V1_HDR_SIZE)

/**
 * MsgVersion v2
 */
// 这个地方对应kafka协议里其实已经不再叫message, 而是叫 record
/*
/*
Record =>
  Length => varint
  Attributes => int8
  TimestampDelta => varint
  OffsetDelta => varint
  KeyLen => varint
  Key => data
  ValueLen => varint
  //Value => data
  Headers => [Header] 
*/
*/
#define RD_KAFKAP_MESSAGE_V2_OVERHEAD                                  \
        (                                                              \
        /* Length (varint) */                                          \
        RD_UVARINT_ENC_SIZEOF(int32_t) +                               \
        /* Attributes */                                               \
        1 +                                                            \
        /* TimestampDelta (varint) */                                  \
        RD_UVARINT_ENC_SIZEOF(int64_t) +                               \
        /* OffsetDelta (varint) */                                     \
        RD_UVARINT_ENC_SIZEOF(int32_t) +                               \
        /* KeyLen (varint) */                                          \
        RD_UVARINT_ENC_SIZEOF(int32_t) +                               \
        /* ValueLen (varint) */                                        \
        RD_UVARINT_ENC_SIZEOF(int32_t) +                               \
        /* HeaderCnt (varint): */                                      \
        RD_UVARINT_ENC_SIZEOF(int32_t)                                 \
        )

/* Old MessageSet header: none */
#define RD_KAFKAP_MSGSET_V0_SIZE                0

/* MessageSet v2 header */
// 对应着 RecordBatch的头:
/*
RecordBatch =>
  FirstOffset => int64
  Length => int32
  PartitionLeaderEpoch => int32
  Magic => int8 
  CRC => int32
  Attributes => int16
  LastOffsetDelta => int32
  FirstTimestamp => int64
  MaxTimestamp => int64
  ProducerId => int64
  ProducerEpoch => int16
  FirstSequence => int32
*/
#define RD_KAFKAP_MSGSET_V2_SIZE                (8+4+4+1+4+2+4+8+8+8+2+4+4)

/* Byte offsets for MessageSet fields */
#define RD_KAFKAP_MSGSET_V2_OF_Length           (8)
#define RD_KAFKAP_MSGSET_V2_OF_CRC              (8+4+4+1)
#define RD_KAFKAP_MSGSET_V2_OF_Attributes       (8+4+4+1+4)
#define RD_KAFKAP_MSGSET_V2_OF_LastOffsetDelta  (8+4+4+1+4+2)
#define RD_KAFKAP_MSGSET_V2_OF_BaseTimestamp    (8+4+4+1+4+2+4)
#define RD_KAFKAP_MSGSET_V2_OF_MaxTimestamp     (8+4+4+1+4+2+4+8)
#define RD_KAFKAP_MSGSET_V2_OF_RecordCount      (8+4+4+1+4+2+4+8+8+8+2+4)
```
###### KafkaApiRequest
* 所在文件：src/rdkafka_proto.h
* 为何需要有这个协议:
   1. kafka协议据有向后兼容的特性，它的同一个reqeust或response为了修复某些bug等原因也可能有多个版本;
   2. 新的kafka broker也可能是增加一些新request的支持，因此需要增加协议让client可以知道当前的broker都支持哪些request;
* 注意事项：
   1. broker针对每种协议会返回所支持的最大版本号和最小版本号;
   2. 客户端从同一集群的多个broker获取的各协议的版本号范围不同，取交集;
   3. 从kafka 0.10.0.0才开始支持此request;
* [kafka 官方 KIP参考](https://cwiki.apache.org/confluence/display/KAFKA/KIP-35+-+Retrieving+protocol+version)
* KafkaApiVersion结构定义:
```
struct rd_kafka_ApiVersion {
	int16_t ApiKey;
	int16_t MinVer;
	int16_t MaxVer;
};
```
* 在 `src/rdkafka_feature.c`中定义了kafka各版本所支持的api即其最大最小版本:
```
/* >= 0.10.0.0: dummy for all future versions that support ApiVersionRequest */
static struct rd_kafka_ApiVersion rd_kafka_ApiVersion_Queryable[] = {
	{ RD_KAFKAP_ApiVersion, 0, 0 }
};
/* =~ 0.9.0 */
static struct rd_kafka_ApiVersion rd_kafka_ApiVersion_0_9_0[]
/* =~ 0.8.2 */
static struct rd_kafka_ApiVersion rd_kafka_ApiVersion_0_8_2[]
/* =~ 0.8.1 */
static struct rd_kafka_ApiVersion rd_kafka_ApiVersion_0_8_1[]
/* =~ 0.8.0 */
static struct rd_kafka_ApiVersion rd_kafka_ApiVersion_0_8_0[]
```
###### Kafka Features
* 通过`KafkaApiRequest`我们可以知道broker目前所支持的协议, 不要忘了,我们的client sdk也是在向前演进的,也有一个协议兼容和支持的问题;
* Librdkafka中通过 `feature map`来表明自己目前所支持kafka的哪些协议的哪些版本, 其支持的 feature map通过 `rd_kafka_feature_map`定义:
```
static const struct rd_kafka_feature_map {
	/* RD_KAFKA_FEATURE_... */
	int feature;

	/* Depends on the following ApiVersions overlapping with
	 * what the broker supports: */
	struct rd_kafka_ApiVersion depends[RD_KAFKAP__NUM];

} rd_kafka_feature_map[]
```
* 根据broker version(<0.10.0)来获取支持的kafka api version信息, 通过`apisp`可以拿到:
```
int rd_kafka_get_legacy_ApiVersions (const char *broker_version,
				     struct rd_kafka_ApiVersion **apisp,
				     size_t *api_cntp, const char *fallback) {
	static const struct {
		const char *pfx;
		struct rd_kafka_ApiVersion *apis;
		size_t api_cnt;
	} vermap[] = {
#define _VERMAP(PFX,APIS) { PFX, APIS, RD_ARRAYSIZE(APIS) }
		_VERMAP("0.9.0", rd_kafka_ApiVersion_0_9_0),
		_VERMAP("0.8.2", rd_kafka_ApiVersion_0_8_2),
		_VERMAP("0.8.1", rd_kafka_ApiVersion_0_8_1),
		_VERMAP("0.8.0", rd_kafka_ApiVersion_0_8_0),
		{ "0.7.", NULL }, /* Unsupported */
		{ "0.6.", NULL }, /* Unsupported */
		_VERMAP("", rd_kafka_ApiVersion_Queryable),
		{ NULL }
	};
	int i;
	int fallback_i = -1;
        int ret = 0;

        *apisp = NULL;
        *api_cntp = 0;

	for (i = 0 ; vermap[i].pfx ; i++) {
                // 比对broker版本
		if (!strncmp(vermap[i].pfx, broker_version, strlen(vermap[i].pfx))) {
			if (!vermap[i].apis)
				return 0;
			*apisp = vermap[i].apis;
			*api_cntp = vermap[i].api_cnt;
                        ret = 1;
                        break;
		} else if (fallback && !strcmp(vermap[i].pfx, fallback))
			fallback_i = i;
	}

	if (!*apisp && fallback) {
		rd_kafka_assert(NULL, fallback_i != -1);
		*apisp    = vermap[fallback_i].apis;
		*api_cntp = vermap[fallback_i].api_cnt;
	}

        return ret;
}
```
* Client kafka Features 检测:
```
int rd_kafka_features_check (rd_kafka_broker_t *rkb,
			     struct rd_kafka_ApiVersion *broker_apis,
			     size_t broker_api_cnt) {
	int features = 0;
	int i;

	/* Scan through features. */
       // 扫描所有已定义的 feature map
	for (i = 0 ; rd_kafka_feature_map[i].feature != 0 ; i++) {
		const struct rd_kafka_ApiVersion *match;
		int fails = 0;

		/* For each feature check that all its API dependencies
		 * can be fullfilled. */

                // 针对每一个feature, 满足其定义的所有依赖的api版本, 才算是支持这个feature
		for (match = &rd_kafka_feature_map[i].depends[0] ;
		     match->ApiKey != -1 ; match++) {
			int r;
			
			r = rd_kafka_ApiVersion_check(broker_apis, broker_api_cnt,
						      match);

			rd_rkb_dbg(rkb, FEATURE, "APIVERSION",
				   " Feature %s: %s (%hd..%hd) "
				   "%ssupported by broker",
				   rd_kafka_features2str(rd_kafka_feature_map[i].
							feature),
				   rd_kafka_ApiKey2str(match->ApiKey),
				   match->MinVer, match->MaxVer,
				   r ? "" : "NOT ");

			fails += !r;
		}

		rd_rkb_dbg(rkb, FEATURE, "APIVERSION",
			   "%s feature %s",
			   fails ? "Disabling" : "Enabling",
			   rd_kafka_features2str(rd_kafka_feature_map[i].feature));


		if (!fails)
			features |= rd_kafka_feature_map[i].feature;
	}

	return features;
}
```
---
### [Librdkafka源码分析-Content Table](https://www.jianshu.com/p/1a94bb09a6e6)
