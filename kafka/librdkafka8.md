* struct rd_kafka_message_t
* struct rd_kafka_msg_t
* struct rd_kafka_msgq_t
* kafka message的协议格式可参考 [官网](https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol#AGuideToTheKafkaProtocol-Messagesets)
---
###### struct rd_kafka_message_s
* 所在文件: src/rdkafka.h
* 生产的数据在application层调用接口后最终会将数据封装成这个结构, 从broker消费下来的数据回调给application层时也会封装成这个结构;
* 定义:
```
typedef struct rd_kafka_message_s {
	rd_kafka_resp_err_t err;   /**< Non-zero for error signaling. */
	rd_kafka_topic_t *rkt;     /**< Topic */
	int32_t partition;         /**< Partition */
	void   *payload;           /**< Producer: original message payload.
				    * Consumer: Depends on the value of \c err :
				    * - \c err==0: Message payload.
				    * - \c err!=0: Error string */
	size_t  len;               /**< Depends on the value of \c err :
				    * - \c err==0: Message payload length
				    * - \c err!=0: Error string length */
	void   *key;               /**< Depends on the value of \c err :
				    * - \c err==0: Optional message key */
	size_t  key_len;           /**< Depends on the value of \c err :
				    * - \c err==0: Optional message key length*/
	int64_t offset;            /**< Consume:
                                    * - Message offset (or offset for error
				    *   if \c err!=0 if applicable).
                                    * - dr_msg_cb:
                                    *   Message offset assigned by broker.
                                    *   If \c produce.offset.report is set then
                                    *   each message will have this field set,
                                    *   otherwise only the last message in
                                    *   each produced internal batch will
                                    *   have this field set, otherwise 0. */
	void  *_private;           /**< Consume:
				    *  - rdkafka private pointer: DO NOT MODIFY
				    *  - dr_msg_cb:
                                    *    msg_opaque from produce() call */
} rd_kafka_message_t;
```
###### struct rd_kafka_msg_t
* 所在文件: src/rdkafka_msg
* 封装了上面的 `struct rd_kafka_message_s`
* 定义:
```
typedef struct rd_kafka_msg_s {
	rd_kafka_message_t rkm_rkmessage;  /* MUST be first field */

        // 使其成为tailq的元素
	TAILQ_ENTRY(rd_kafka_msg_s)  rkm_link;

	int        rkm_flags;

        // 时间戳, 分两类: 客户端生间时的时间和broker接收后作append log时的时间
	int64_t    rkm_timestamp;  
	rd_kafka_timestamp_type_t rkm_tstype; /* rkm_timestamp type */

        union {
                struct {
                        rd_ts_t ts_timeout; /* Message timeout */
                        rd_ts_t ts_enq;     /* Enqueue/Produce time */
                } producer;
        } rkm_u;
} rd_kafka_msg_t;
```
* `rd_kafka_message_t`类型转化为`rd_kafka_msg_t`:
`rd_kafka_message_t rkm_rkmessage`必须是`struct rd_kafka_msg_s`结构的第一个字段,
```
rd_kafka_msg_t *rd_kafka_message2msg (rd_kafka_message_t *rkmessage) {
	return (rd_kafka_msg_t *)rkmessage;
}
```
* 发送kafka message前的审计
librdkafka支持异步发送, 本地有发送缓冲区, 因为在发送前需要作check, 看发送队列是否已满, 如果设置了block发送, 在发送队列满的情况在要一直阻塞wait, 直到被signal
```
static RD_INLINE RD_UNUSED rd_kafka_resp_err_t
rd_kafka_curr_msgs_add (rd_kafka_t *rk, unsigned int cnt, size_t size,
			int block) {

	if (rk->rk_type != RD_KAFKA_PRODUCER)
		return RD_KAFKA_RESP_ERR_NO_ERROR;

	mtx_lock(&rk->rk_curr_msgs.lock);
	while (unlikely(rk->rk_curr_msgs.cnt + cnt >
			rk->rk_curr_msgs.max_cnt ||
			(unsigned long long)(rk->rk_curr_msgs.size + size) >
			(unsigned long long)rk->rk_curr_msgs.max_size)) {
		if (!block) {
			mtx_unlock(&rk->rk_curr_msgs.lock);
			return RD_KAFKA_RESP_ERR__QUEUE_FULL;
		}

		cnd_wait(&rk->rk_curr_msgs.cnd, &rk->rk_curr_msgs.lock);
	}

	rk->rk_curr_msgs.cnt  += cnt;
	rk->rk_curr_msgs.size += size;
	mtx_unlock(&rk->rk_curr_msgs.lock);

	return RD_KAFKA_RESP_ERR_NO_ERROR;
}
```
* 创建rd_kafka_msg_t, 内部接口 `rd_kafka_msg_t *rd_kafka_msg_new00`
```
rd_kafka_msg_t *rd_kafka_msg_new00 (rd_kafka_itopic_t *rkt,
				    int32_t partition,
				    int msgflags,
				    char *payload, size_t len,
				    const void *key, size_t keylen,
				    void *msg_opaque) {
	rd_kafka_msg_t *rkm;
	size_t mlen = sizeof(*rkm);
	char *p;

	/* If we are to make a copy of the payload, allocate space for it too */
        // 如果设置了RD_KAFKA_MSG_F_COPY,  需要为payload分配内存,在rd_kafka_msg_t后面
	if (msgflags & RD_KAFKA_MSG_F_COPY) {
		msgflags &= ~RD_KAFKA_MSG_F_FREE;
		mlen += len;
	}

	mlen += keylen;

	/* Note: using rd_malloc here, not rd_calloc, so make sure all fields
	 *       are properly set up. */
	rkm                 = rd_malloc(mlen);
	rkm->rkm_err        = 0;
	rkm->rkm_flags      = RD_KAFKA_MSG_F_FREE_RKM | msgflags;
	rkm->rkm_len        = len;
	rkm->rkm_opaque     = msg_opaque;
	rkm->rkm_rkmessage.rkt = rd_kafka_topic_keep_a(rkt);

	rkm->rkm_partition  = partition;
        rkm->rkm_offset     = RD_KAFKA_OFFSET_INVALID;
	rkm->rkm_timestamp  = 0;
	rkm->rkm_tstype     = RD_KAFKA_TIMESTAMP_NOT_AVAILABLE;

	p = (char *)(rkm+1);

        // 复制payload
	if (payload && msgflags & RD_KAFKA_MSG_F_COPY) {
		/* Copy payload to space following the ..msg_t */
		rkm->rkm_payload = p;
		memcpy(rkm->rkm_payload, payload, len);
		p += len;

	} else {
		/* Just point to the provided payload. */
		rkm->rkm_payload = payload;
	}

	if (key) {
		rkm->rkm_key     = p;
		rkm->rkm_key_len = keylen;
		memcpy(rkm->rkm_key, key, keylen);
	} else {
		rkm->rkm_key = NULL;
		rkm->rkm_key_len = 0;
	}

        return rkm;
}
```
* 创建rd_kafka_msg_t, 创建之前增加check, 内部接口 `rd_kafka_msg_t *rd_kafka_msg_new0`
```
static rd_kafka_msg_t *rd_kafka_msg_new0 (rd_kafka_itopic_t *rkt,
                                          int32_t force_partition,
                                          int msgflags,
                                          char *payload, size_t len,
                                          const void *key, size_t keylen,
                                          void *msg_opaque,
                                          rd_kafka_resp_err_t *errp,
                                          int *errnop,
                                          int64_t timestamp,
                                          rd_ts_t now) {
	rd_kafka_msg_t *rkm;

	if (unlikely(!payload))
		len = 0;
	if (!key)
		keylen = 0;

        // 检查msg大小是否超出了配置的最大msg大小
	if (unlikely(len + keylen >
		     (size_t)rkt->rkt_rk->rk_conf.max_msg_size ||
		     keylen > INT32_MAX)) {
		*errp = RD_KAFKA_RESP_ERR_MSG_SIZE_TOO_LARGE;
		if (errnop)
			*errnop = EMSGSIZE;
		return NULL;
	}

        // 检查发送队列是否已满, 如果设置了block发送, 在发送队列满的情况在要一直阻塞wait, 直到被signal
	*errp = rd_kafka_curr_msgs_add(rkt->rkt_rk, 1, len,
				       msgflags & RD_KAFKA_MSG_F_BLOCK);
	if (unlikely(*errp)) {
		if (errnop)
			*errnop = ENOBUFS;
		return NULL;
	}

        // 创建 rd_kakfa_msg_t
	rkm = rd_kafka_msg_new00(rkt, force_partition,
				 msgflags|RD_KAFKA_MSG_F_ACCOUNT /* curr_msgs_add() */,
				 payload, len, key, keylen, msg_opaque);

        if (timestamp)
                rkm->rkm_timestamp  = timestamp;
        else
                rkm->rkm_timestamp = rd_uclock()/1000;
        rkm->rkm_tstype     = RD_KAFKA_TIMESTAMP_CREATE_TIME;

        rkm->rkm_ts_enq = now;

	if (rkt->rkt_conf.message_timeout_ms == 0) {
		rkm->rkm_ts_timeout = INT64_MAX;
	} else {
		rkm->rkm_ts_timeout = now +
			rkt->rkt_conf.message_timeout_ms * 1000;
	}

        /* Call interceptor chain for on_send */
        // on_send拦截器, 对这个rkm->rkm_rkmessage作一些个性化处理
        rd_kafka_interceptors_on_send(rkt->rkt_rk, &rkm->rkm_rkmessage);

        return rkm;
}
```
* 创建rd_kafka_msg_t,   并放入选定的topic-partition的队列
```
int rd_kafka_msg_new (rd_kafka_itopic_t *rkt, int32_t force_partition,
		      int msgflags,
		      char *payload, size_t len,
		      const void *key, size_t keylen,
		      void *msg_opaque) {
	rd_kafka_msg_t *rkm;
	rd_kafka_resp_err_t err;
	int errnox;

        /* Create message */
        // 创建 rd_kafka_msg_t
        rkm = rd_kafka_msg_new0(rkt, force_partition, msgflags, 
                                payload, len, key, keylen, msg_opaque,
                                &err, &errnox, 0, rd_clock());
        if (unlikely(!rkm)) {
                /* errno is already set by msg_new() */
		rd_kafka_set_last_error(err, errnox);
                return -1;
        }


        /* Partition the message */
       // 选定topic-parition, 放入队列, 这个函数很重要, 我们会单独讲
	err = rd_kafka_msg_partitioner(rkt, rkm, 1);
	if (likely(!err)) {
		rd_kafka_set_last_error(0, 0);
		return 0;
	}

       // 失败的话, 作清理, 设置error
        /* Interceptor: unroll failing messages by triggering on_ack.. */
        rkm->rkm_err = err;
        rd_kafka_interceptors_on_acknowledgement(rkt->rkt_rk,
                                                 &rkm->rkm_rkmessage);

	/* Handle partitioner failures: it only fails when the application
	 * attempts to force a destination partition that does not exist
	 * in the cluster.  Note we must clear the RD_KAFKA_MSG_F_FREE
	 * flag since our contract says we don't free the payload on
	 * failure. */

	rkm->rkm_flags &= ~RD_KAFKA_MSG_F_FREE;
	rd_kafka_msg_destroy(rkt->rkt_rk, rkm);

	/* Translate error codes to errnos. */
	if (err == RD_KAFKA_RESP_ERR__UNKNOWN_PARTITION)
		rd_kafka_set_last_error(err, ESRCH);
	else if (err == RD_KAFKA_RESP_ERR__UNKNOWN_TOPIC)
		rd_kafka_set_last_error(err, ENOENT);
	else
		rd_kafka_set_last_error(err, EINVAL); /* NOTREACHED */

	return -1;
}
```
###### struct rd_kafka_msgq_t
* 所在文件: src/rdkafka_msg.h
* 其实就是简单封装的`rd_kafka_msg_t`队列
* 定义:
```
typedef struct rd_kafka_msgq_s {
	TAILQ_HEAD(, rd_kafka_msg_s) rkmq_msgs;

        // kafka message 个数
	rd_atomic32_t rkmq_msg_cnt;

        // kafka message总大小
	rd_atomic64_t rkmq_msg_bytes;
} rd_kafka_msgq_t;
```
* 合并两个`rd_kafka_msgq_t`: 
```
void rd_kafka_msgq_concat (rd_kafka_msgq_t *dst,
						   rd_kafka_msgq_t *src)
```
* 使用一个`rd_kafka_msgq_t`覆盖另一个`rd_kafka_msgq_t`
```
void rd_kafka_msgq_move (rd_kafka_msgq_t *dst,
						 rd_kafka_msgq_t *src)
```
* 从队列里删除一个`rd_kafka_msgq_t`
```
rd_kafka_msg_t *rd_kafka_msgq_deq (rd_kafka_msgq_t *rkmq,
				   rd_kafka_msg_t *rkm,
				   int do_count)
```
* 出队列
```
rd_kafka_msg_t *rd_kafka_msgq_pop (rd_kafka_msgq_t *rkmq) {
	rd_kafka_msg_t *rkm;

	if (((rkm = TAILQ_FIRST(&rkmq->rkmq_msgs))))
		rd_kafka_msgq_deq(rkmq, rkm, 1);

	return rkm;
}
```
* 入队列, 插入到队尾
```
static RD_INLINE RD_UNUSED void rd_kafka_msgq_enq (rd_kafka_msgq_t *rkmq,
						rd_kafka_msg_t *rkm) {
	TAILQ_INSERT_TAIL(&rkmq->rkmq_msgs, rkm, rkm_link);
	rd_atomic32_add(&rkmq->rkmq_msg_cnt, 1);
	rd_atomic64_add(&rkmq->rkmq_msg_bytes, rkm->rkm_len+rkm->rkm_key_len);
}
```
* 扫描队列, 将超时的加入到超时队列
```
int rd_kafka_msgq_age_scan (rd_kafka_msgq_t *rkmq,
			    rd_kafka_msgq_t *timedout,
			    rd_ts_t now) {
	rd_kafka_msg_t *rkm, *tmp;
	int cnt = rd_atomic32_get(&timedout->rkmq_msg_cnt);

	/* Assume messages are added in time sequencial order */
	TAILQ_FOREACH_SAFE(rkm, &rkmq->rkmq_msgs, rkm_link, tmp) {
		if (likely(rkm->rkm_ts_timeout > now))
			break;

		rd_kafka_msgq_deq(rkmq, rkm, 1);
		rd_kafka_msgq_enq(timedout, rkm);
	}

	return rd_atomic32_get(&timedout->rkmq_msg_cnt) - cnt;
}
```
* `rd_kafka_msg_partitioner`很重要的一个函数, 作两件事: 确定一个topic-partition, 然后把这个`rd_kafka_msgq_t`放到这个topic-parition对应的队列里
```
int rd_kafka_msg_partitioner (rd_kafka_itopic_t *rkt, rd_kafka_msg_t *rkm,
			      int do_lock) {
	int32_t partition;
	rd_kafka_toppar_t *rktp_new;
        shptr_rd_kafka_toppar_t *s_rktp_new;
	rd_kafka_resp_err_t err;

	if (do_lock)
		rd_kafka_topic_rdlock(rkt);

       // 根据这个topic当前的状态, 分别作处理
        switch (rkt->rkt_state)
        {
        case RD_KAFKA_TOPIC_S_UNKNOWN:
                /* No metadata received from cluster yet.
                 * Put message in UA partition and re-run partitioner when
                 * cluster comes up. */
		partition = RD_KAFKA_PARTITION_UA;
                break;

        case RD_KAFKA_TOPIC_S_NOTEXISTS:
                /* Topic not found in cluster.
                 * Fail message immediately. */
                err = RD_KAFKA_RESP_ERR__UNKNOWN_TOPIC;
		if (do_lock)
			rd_kafka_topic_rdunlock(rkt);
                return err;

        case RD_KAFKA_TOPIC_S_EXISTS:
                /* Topic exists in cluster. */

                /* Topic exists but has no partitions.
                 * This is usually an transient state following the
                 * auto-creation of a topic. */
                if (unlikely(rkt->rkt_partition_cnt == 0)) {
                        partition = RD_KAFKA_PARTITION_UA;
                        break;
                }

                /* Partition not assigned, run partitioner. */
                // 如果rkm->rkm_partition == RD_KAFKA_PARTITION_UA, 调用设转置的partitioner函数来确定一个partition
                if (rkm->rkm_partition == RD_KAFKA_PARTITION_UA) {
                        rd_kafka_topic_t *app_rkt;
                        /* Provide a temporary app_rkt instance to protect
                         * from the case where the application decided to
                         * destroy its topic object prior to delivery completion
                         * (issue #502). */
                        app_rkt = rd_kafka_topic_keep_a(rkt);
                        partition = rkt->rkt_conf.
                                partitioner(app_rkt,
                                            rkm->rkm_key,
					    rkm->rkm_key_len,
                                            rkt->rkt_partition_cnt,
                                            rkt->rkt_conf.opaque,
                                            rkm->rkm_opaque);
                        rd_kafka_topic_destroy0(
                                rd_kafka_topic_a2s(app_rkt));
                } else
                        partition = rkm->rkm_partition;

                /* Check that partition exists. */
                if (partition >= rkt->rkt_partition_cnt) {
                        err = RD_KAFKA_RESP_ERR__UNKNOWN_PARTITION;
                        if (do_lock)
                                rd_kafka_topic_rdunlock(rkt);
                        return err;
                }
                break;

        default:
                rd_kafka_assert(rkt->rkt_rk, !*"NOTREACHED");
                break;
        }

	/* Get new partition */
	s_rktp_new = rd_kafka_toppar_get(rkt, partition, 0);

	if (unlikely(!s_rktp_new)) {
		/* Unknown topic or partition */
		if (rkt->rkt_state == RD_KAFKA_TOPIC_S_NOTEXISTS)
			err = RD_KAFKA_RESP_ERR__UNKNOWN_TOPIC;
		else
			err = RD_KAFKA_RESP_ERR__UNKNOWN_PARTITION;

		if (do_lock)
			rd_kafka_topic_rdunlock(rkt);

		return  err;
	}

        rktp_new = rd_kafka_toppar_s2i(s_rktp_new);
        rd_atomic64_add(&rktp_new->rktp_c.msgs, 1);

        /* Update message partition */
        if (rkm->rkm_partition == RD_KAFKA_PARTITION_UA)
                rkm->rkm_partition = partition;

	/* Partition is available: enqueue msg on partition's queue */
        // 塞到partition队列的队尾
	rd_kafka_toppar_enq_msg(rktp_new, rkm);
	if (do_lock)
		rd_kafka_topic_rdunlock(rkt);
	rd_kafka_toppar_destroy(s_rktp_new); /* from _get() */
	return 0;
}
```
---
### [Librdkafka源码分析-Content Table](https://www.jianshu.com/p/1a94bb09a6e6)