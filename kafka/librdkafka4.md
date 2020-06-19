下面要介绍的数据类型都是在kafka protocol的序列化中使用的
* Kafka Protocol String
* Kafka Protocol ByteArray
---
######  Kafka Protocol String
* 所在文件：src/rdkafka_proto.h
* 表示kafka协议中的字符串，在协议的序列化中，先用2个字节表示字符串内容的长度，不包含结尾的\0, 紧随其后是字符串的内容: `{ uint16, data.. }`
* 定义如下，包含长度和指向实际字符内容的指针;
```
typedef struct rd_kafkap_str_s {
	/* convenience header (aligned access, host endian) */
	int         len; /* Kafka string length (-1=NULL, 0=empty, >0=string) */

	const char *str; /* points into data[] or other memory,
			  * not NULL-terminated */
} rd_kafkap_str_t;
```
可以表示三种字符串格式：
  1. len = -1, 是一个null字符串，str = null;
  2. len = 0, 是一个空字符串， str = "";
  3. len > 0, 是一个长度 为len的字符串， 这个len不包含结尾的\0  ;

* 获取Kafka String长度，根据`rd_kafkap_str_t::len`来判断
```
#define RD_KAFKAP_STR_LEN0(len) ((len) == RD_KAFKAP_STR_LEN_NULL ? 0 : (len))
#define RD_KAFKAP_STR_LEN(kstr) RD_KAFKAP_STR_LEN0((kstr)->len)
```
* 获取Kafka String序列化后的长度，即在TCP发送协议中的长度：
```
/* Returns the actual size of a kafka protocol string representation. */
#define RD_KAFKAP_STR_SIZE0(len) (2 + RD_KAFKAP_STR_LEN0(len))
#define RD_KAFKAP_STR_SIZE(kstr) RD_KAFKAP_STR_SIZE0((kstr)->len)
```
上面的`2`字节用来放字串的长度
* Kafka Protocol String的创建：不光要创建一个`rd_kafkap_str_t`对象，还要在其内存后紧挨着创建序列化所需要的内存空间，具体看下面代码里的注释
```
static RD_INLINE RD_UNUSED
rd_kafkap_str_t *rd_kafkap_str_new (const char *str, int len) {
	rd_kafkap_str_t *kstr;
	int16_t klen;

	if (!str)
		len = RD_KAFKAP_STR_LEN_NULL;
	else if (len == -1)
		len = str ? (int)strlen(str) : RD_KAFKAP_STR_LEN_NULL;

        //`rd_kafkap_str_t`结构体大小 + 2字节存放字符串长度 + 字符串实际长度 + 1字节的字符串结尾\0的长度
	kstr = rd_malloc(sizeof(*kstr) + 2 +
			 (len == RD_KAFKAP_STR_LEN_NULL ? 0 : len + 1));
	kstr->len = len;

	/* 
           Serialised format: 16-bit string length 
           填充2字节的序列化后字符串长度
        */
	klen = htobe16(len);
	memcpy(kstr+1, &klen, 2);

	/* Serialised format: non null-terminated string */
	if (len == RD_KAFKAP_STR_LEN_NULL)
		kstr->str = NULL;
	else {
                // rd_kafkap_str_t::src指向实际的内存地址，copy实际字符串的内容
		kstr->str = ((const char *)(kstr+1))+2;
		memcpy((void *)kstr->str, str, len);
		((char *)kstr->str)[len] = '\0';
	}

	return kstr;
}
```
###### Kafka Protocol Byte Array
* 所在文件：src/rdkafka_proto.h
* 与上在介结的String很相似，表示kafka协议中的字节娄组，在协议的序列化中，先用4个字节表示字节数组的内容的长度，紧随其后是其实际的内容: { uint32, data.. }
* 定义如下：
```
typedef struct rd_kafkap_bytes_s {
	/* convenience header (aligned access, host endian) */
	int32_t     len;   /* Kafka bytes length (-1=NULL, 0=empty, >0=data) */
	const void *data;  /* points just past the struct, or other memory,
			    * not NULL-terminated */
	const char _data[1]; /* Bytes following struct when new()ed */
} rd_kafkap_bytes_t;
```
可以表示三种byte数组：
  1. Kafka NULL bytes (bytes==NULL,len==0),
  2. Empty bytes (bytes!=NULL,len==0)
  3. 有实际数据 data (bytes!=NULL,len>0)
* Kafka Byte Array的创建：
```
static RD_INLINE RD_UNUSED
rd_kafkap_bytes_t *rd_kafkap_bytes_new (const char *bytes, int32_t len) {
	rd_kafkap_bytes_t *kbytes;
	int32_t klen;

	if (!bytes && !len)
		len = RD_KAFKAP_BYTES_LEN_NULL;

        //`rd_kafkap_bytes_t`结构体大小 + 4字节存放byte array长度 + 内存实际长度
	kbytes = rd_malloc(sizeof(*kbytes) + 4 +
			 (len == RD_KAFKAP_BYTES_LEN_NULL ? 0 : len));
	kbytes->len = len;

	klen = htobe32(len);
	memcpy(kbytes+1, &klen, 4);

	if (len == RD_KAFKAP_BYTES_LEN_NULL)
		kbytes->data = NULL;
	else {
		kbytes->data = ((const char *)(kbytes+1))+4;
                if (bytes)
                        memcpy((void *)kbytes->data, bytes, len);
	}

	return kbytes;
}
```

---
### [Librdkafka源码分析-Content Table](https://www.jianshu.com/p/1a94bb09a6e6)

