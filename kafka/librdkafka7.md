* 上一节我们介绍了[Librdkafka中的任务处理队列的相关操作](https://www.jianshu.com/p/4115ee52dcc7), 这一节我们介绍一下放入这个队列中的各种任务(也可以叫event, 也可以叫operator), 也就是各种不同类型的operator
* 具体的op如何处理, 我们会在后期分析具体的流程时再作深入讨论
---
###### struct rd_kafka_op_s
* 所在文件: src/rdkafka_op.h(c)
* 定义:
```
struct rd_kafka_op_s {
        // 加上tailq的元素域
	TAILQ_ENTRY(rd_kafka_op_s) rko_link;

       // op的类型 
	rd_kafka_op_type_t    rko_type;   /* Internal op type */
	rd_kafka_event_type_t rko_evtype;
	int                   rko_flags;  /* See RD_KAFKA_OP_F_... above */

        // 版本控制
	int32_t               rko_version;
	rd_kafka_resp_err_t   rko_err;
	int32_t               rko_len;    /* Depends on type, typically the
					   * message length. */
        // op有优先级之分
        rd_kafka_op_prio_t    rko_prio;   /* In-queue priority.
                                           * Higher value means higher prio. */

        // 所关联的topic-patition, 是个带引用计数的指针
	shptr_rd_kafka_toppar_t *rko_rktp;

        /*
	 * Generic fields
	 */

	/* Indicates request: enqueue reply on rko_replyq.q with .version.
	 * .q is refcounted. */
        // 这个op任务完成后, 执行结果放入哪个队列rd_kafka_replyq_t结构包括一个rd_kafka_q_t指针和这个version字段
	rd_kafka_replyq_t rko_replyq;

        /* Original queue's op serve callback and opaque, if any.
         * Mainly used for forwarded queues to use the original queue's
         * serve function from the forwarded position. */
       // 默认处理函数
        rd_kafka_q_serve_cb_t *rko_serve;
        void *rko_serve_opaque;

        // 相关联的rd_kafka_t句柄
	rd_kafka_t     *rko_rk;

#if ENABLE_DEVEL
        const char *rko_source;  /**< Where op was created */
#endif

        /* RD_KAFKA_OP_CB */
        rd_kafka_op_cb_t *rko_op_cb;

//一个 union类型, 不同的op类型需要各自特定的数据结构,统一定义在这个 union中
union {
struct {
			rd_kafka_buf_t *rkbuf;
			rd_kafka_msg_t  rkm;
			int evidx;
		} fetch;

		struct {
			rd_kafka_topic_partition_list_t *partitions;
			int do_free; /* free .partitions on destroy() */
		} offset_fetch;
               ...
}
}
```
* 创建`rd_kafka_op_s`:
```
rd_kafka_op_t *rd_kafka_op_new0 (const char *source, rd_kafka_op_type_t type) {
	rd_kafka_op_t *rko;
       // 每种op特有的user data的结构大小
        static const size_t op2size[RD_KAFKA_OP__END] = {
                [RD_KAFKA_OP_FETCH] = sizeof(rko->rko_u.fetch),
                [RD_KAFKA_OP_ERR] = sizeof(rko->rko_u.err),
                [RD_KAFKA_OP_CONSUMER_ERR] = sizeof(rko->rko_u.err),
                [RD_KAFKA_OP_DR] = sizeof(rko->rko_u.dr),
                [RD_KAFKA_OP_STATS] = sizeof(rko->rko_u.stats),
                [RD_KAFKA_OP_OFFSET_COMMIT] = sizeof(rko->rko_u.offset_commit),
                [RD_KAFKA_OP_NODE_UPDATE] = sizeof(rko->rko_u.node),
                [RD_KAFKA_OP_XMIT_BUF] = sizeof(rko->rko_u.xbuf),
                [RD_KAFKA_OP_RECV_BUF] = sizeof(rko->rko_u.xbuf),
                [RD_KAFKA_OP_XMIT_RETRY] = sizeof(rko->rko_u.xbuf),
                [RD_KAFKA_OP_FETCH_START] = sizeof(rko->rko_u.fetch_start),
                [RD_KAFKA_OP_FETCH_STOP] = 0,
                [RD_KAFKA_OP_SEEK] = sizeof(rko->rko_u.fetch_start),
                [RD_KAFKA_OP_PAUSE] = sizeof(rko->rko_u.pause),
                [RD_KAFKA_OP_OFFSET_FETCH] = sizeof(rko->rko_u.offset_fetch),
                [RD_KAFKA_OP_PARTITION_JOIN] = 0,
                [RD_KAFKA_OP_PARTITION_LEAVE] = 0,
                [RD_KAFKA_OP_REBALANCE] = sizeof(rko->rko_u.rebalance),
                [RD_KAFKA_OP_TERMINATE] = 0,
                [RD_KAFKA_OP_COORD_QUERY] = 0,
                [RD_KAFKA_OP_SUBSCRIBE] = sizeof(rko->rko_u.subscribe),
                [RD_KAFKA_OP_ASSIGN] = sizeof(rko->rko_u.assign),
                [RD_KAFKA_OP_GET_SUBSCRIPTION] = sizeof(rko->rko_u.subscribe),
                [RD_KAFKA_OP_GET_ASSIGNMENT] = sizeof(rko->rko_u.assign),
                [RD_KAFKA_OP_THROTTLE] = sizeof(rko->rko_u.throttle),
                [RD_KAFKA_OP_NAME] = sizeof(rko->rko_u.name),
                [RD_KAFKA_OP_OFFSET_RESET] = sizeof(rko->rko_u.offset_reset),
                [RD_KAFKA_OP_METADATA] = sizeof(rko->rko_u.metadata),
                [RD_KAFKA_OP_LOG] = sizeof(rko->rko_u.log),
                [RD_KAFKA_OP_WAKEUP] = 0,
	};
	size_t tsize = op2size[type & ~RD_KAFKA_OP_FLAGMASK];

        // 分配内存
	rko = rd_calloc(1, sizeof(*rko)-sizeof(rko->rko_u)+tsize);
	rko->rko_type = type;

#if ENABLE_DEVEL
        rko->rko_source = source;
        rd_atomic32_add(&rd_kafka_op_cnt, 1);
#endif
	return rko;
}
```
* 产生一个error事情`rd_kafka_q_op_err`:
```
void rd_kafka_q_op_err (rd_kafka_q_t *rkq, rd_kafka_op_type_t optype,
                        rd_kafka_resp_err_t err, int32_t version,
			rd_kafka_toppar_t *rktp, int64_t offset,
                        const char *fmt, ...) {
	va_list ap;
	char buf[2048];
	rd_kafka_op_t *rko;

	va_start(ap, fmt);
	rd_vsnprintf(buf, sizeof(buf), fmt, ap);
	va_end(ap);

        //创建op并赋值
	rko = rd_kafka_op_new(optype);
	rko->rko_version = version;
	rko->rko_err = err;
	rko->rko_u.err.offset = offset;
	rko->rko_u.err.errstr = rd_strdup(buf);
	if (rktp)
		rko->rko_rktp = rd_kafka_toppar_keep(rktp);

       // 放入队列
	rd_kafka_q_enq(rkq, rko);
}
```
* 将一个表示request的op放入队列并等待response, `rd_kafka_op_t *rd_kafka_op_req0`:
```
rd_kafka_op_t *rd_kafka_op_req0 (rd_kafka_q_t *destq,
                                 rd_kafka_q_t *recvq,
                                 rd_kafka_op_t *rko,
                                 int timeout_ms) {
        rd_kafka_op_t *reply;

        /* Indicate to destination where to send reply. */
       //设置放response的队列
        rd_kafka_op_set_replyq(rko, recvq, NULL);

        /* Enqueue op */
       // request进队列
        if (!rd_kafka_q_enq(destq, rko))
                return NULL;

        /* Wait for reply */
       // 等待request处理完成
        reply = rd_kafka_q_pop(recvq, timeout_ms, 0);

        /* May be NULL for timeout */
        return reply;
}
```
* op处理`rd_kafka_op_handle `:
```
rd_kafka_op_res_t
rd_kafka_op_handle (rd_kafka_t *rk, rd_kafka_q_t *rkq, rd_kafka_op_t *rko,
                    rd_kafka_q_cb_type_t cb_type, void *opaque,
                    rd_kafka_q_serve_cb_t *callback) {
        rd_kafka_op_res_t res;

       // 先使用  rd_kafka_op_handle_std处理
        res = rd_kafka_op_handle_std(rk, rkq, rko, cb_type);
        if (res == RD_KAFKA_OP_RES_HANDLED) {
                rd_kafka_op_destroy(rko);
                return res;
        } else if (unlikely(res == RD_KAFKA_OP_RES_YIELD))
                return res;

       // 如果rko设置了回调, 则调用其回调
        if (rko->rko_serve) {
                callback = rko->rko_serve;
                opaque   = rko->rko_serve_opaque;
                rko->rko_serve        = NULL;
                rko->rko_serve_opaque = NULL;
        }

        if (callback)
                res = callback(rk, rkq, rko, cb_type, opaque);

        return res;
}
```
---
### [Librdkafka源码分析-Content Table](https://www.jianshu.com/p/1a94bb09a6e6)