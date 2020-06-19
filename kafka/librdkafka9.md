* topic-partition是kafka分布式的精华, 也是针对kafka进行生产或消费的最小单元;
* 在这篇里我们开始介绍相关的数据结构
* 内容如下:
   1. rd_kafka_topic_partition_t
   2. rd_kafka_topic_partition_list_t
   3. rd_kafka_toppar_s
---
###### rd_kafka_topic_partition_t
* 所在文件: src/rdkafka.h
* 定义了一个partition的相关数据结构, 简单定义, 占位符
* 定义:
```
typedef struct rd_kafka_topic_partition_s {
        char        *topic;             /**< Topic name */
        int32_t      partition;         /**< Partition */
	int64_t      offset;            /**< Offset */
        void        *metadata;          /**< Metadata */ // 主要是leader, replicas, isr等信息
        size_t       metadata_size;     /**< Metadata size */
        void        *opaque;            /**< Application opaque */
        rd_kafka_resp_err_t err;        /**< Error code, depending on use. */
        void       *_private;           /**< INTERNAL USE ONLY,
                                         *   INITIALIZE TO ZERO, DO NOT TOUCH */
} rd_kafka_topic_partition_t;

```
###### rd_kafka_topic_partition_list_t
* 所在文件: src/rdkafka.h
* 用来存储 `rd_kafka_topic_partition_t`的可动态扩容的数组
* 定义:
```
typedef struct rd_kafka_topic_partition_list_s {
        int cnt;               /**< Current number of elements */ 当前数组中放入的element数量
        int size;              /**< Current allocated size */ // 当前数组的容量
        rd_kafka_topic_partition_t *elems; /**< Element array[] */ 动态数组指针
} rd_kafka_topic_partition_list_t;
```
* 扩容操作 `rd_kafka_topic_partition_list_grow`:
```
rd_kafka_topic_partition_list_grow (rd_kafka_topic_partition_list_t *rktparlist,
                                    int add_size) {
        if (add_size < rktparlist->size)
                add_size = RD_MAX(rktparlist->size, 32);

        rktparlist->size += add_size;
        // 使用realloc重新分配内存
        rktparlist->elems = rd_realloc(rktparlist->elems,
                                       sizeof(*rktparlist->elems) *
                                       rktparlist->size);

}
```
* 创建操作 `rd_kafka_topic_partition_list_new`:
```
rd_kafka_topic_partition_list_t *rd_kafka_topic_partition_list_new (int size) {
        rd_kafka_topic_partition_list_t *rktparlist;
        rktparlist = rd_calloc(1, sizeof(*rktparlist));
        rktparlist->size = size;
        rktparlist->cnt = 0;
        if (size > 0)
                rd_kafka_topic_partition_list_grow(rktparlist, size);
        return rktparlist;
}
```
* 查找操作 `rd_kafka_topic_partition_list_find`: 
topic和partition都相等才算是相等
```
rd_kafka_topic_partition_list_find (rd_kafka_topic_partition_list_t *rktparlist,
				     const char *topic, int32_t partition) {
	int i = rd_kafka_topic_partition_list_find0(rktparlist,
						    topic, partition);
	if (i == -1)
		return NULL;
	else
		return &rktparlist->elems[i];
}
```
* 按索引删除 `rd_kafka_topic_partition_list_del_by_idx`
```
rd_kafka_topic_partition_list_del_by_idx (rd_kafka_topic_partition_list_t *rktparlist,
					  int idx) {
	if (unlikely(idx < 0 || idx >= rktparlist->cnt))
		return 0;

        // element数量减1
	rktparlist->cnt--;
        // destory 删除的元素 
	rd_kafka_topic_partition_destroy0(&rktparlist->elems[idx], 0);

        // 作内存的移动, 但不回收
	memmove(&rktparlist->elems[idx], &rktparlist->elems[idx+1],
		(rktparlist->cnt - idx) * sizeof(rktparlist->elems[idx]));

	return 1;
}
```
* 排序`rd_kafka_topic_partition_list_sort_by_topic`
topic名字不同按topic名字排,topic名字相同按partition排
```
void rd_kafka_topic_partition_list_sort_by_topic (
        rd_kafka_topic_partition_list_t *rktparlist) {
        rd_kafka_topic_partition_list_sort(rktparlist,
                                           rd_kafka_topic_partition_cmp, NULL);
}
```
###### rd_kafka_toppar_s
* 所在文件: src/rdkafka_partition.h
* 重量数据结构,topic, partition, leader, 生产, 消费, 各种定时timer都在里面
* 定义, 这个结构体巨庞大
```
struct rd_kafka_toppar_s { /* rd_kafka_toppar_t */
	TAILQ_ENTRY(rd_kafka_toppar_s) rktp_rklink;  /* rd_kafka_t link */
	TAILQ_ENTRY(rd_kafka_toppar_s) rktp_rkblink; /* rd_kafka_broker_t link*/
        CIRCLEQ_ENTRY(rd_kafka_toppar_s) rktp_fetchlink; /* rkb_fetch_toppars */
	TAILQ_ENTRY(rd_kafka_toppar_s) rktp_rktlink; /* rd_kafka_itopic_t link*/
        TAILQ_ENTRY(rd_kafka_toppar_s) rktp_cgrplink;/* rd_kafka_cgrp_t link */
        rd_kafka_itopic_t       *rktp_rkt;
        shptr_rd_kafka_itopic_t *rktp_s_rkt;  /* shared pointer for rktp_rkt */
	int32_t            rktp_partition;
        //LOCK: toppar_lock() + topic_wrlock()
        //LOCK: .. in partition_available()
        int32_t            rktp_leader_id;   /**< Current leader broker id.
                                              *   This is updated directly
                                              *   from metadata. */
	rd_kafka_broker_t *rktp_leader;      /**< Current leader broker
                                              *   This updated asynchronously
                                              *   by issuing JOIN op to
                                              *   broker thread, so be careful
                                              *   in using this since it
                                              *   may lag. */
        rd_kafka_broker_t *rktp_next_leader; /**< Next leader broker after
                                              *   async migration op. */
	rd_refcnt_t        rktp_refcnt;
	mtx_t              rktp_lock;
 rd_atomic32_t      rktp_version;         /* Latest op version.
                                                  * Authoritative (app thread)*/
	int32_t            rktp_op_version;      /* Op version of curr command
						  * state from.
						  * (broker thread) */
        int32_t            rktp_fetch_version;   /* Op version of curr fetch.
                                                    (broker thread) */

	enum {
		RD_KAFKA_TOPPAR_FETCH_NONE = 0,
                RD_KAFKA_TOPPAR_FETCH_STOPPING,
                RD_KAFKA_TOPPAR_FETCH_STOPPED,
		RD_KAFKA_TOPPAR_FETCH_OFFSET_QUERY,
		RD_KAFKA_TOPPAR_FETCH_OFFSET_WAIT,
		RD_KAFKA_TOPPAR_FETCH_ACTIVE,
	} rktp_fetch_state;    
int32_t            rktp_fetch_msg_max_bytes; /* Max number of bytes to
                                                      * fetch.
                                                      * Locality: broker thread
                                                      */

        rd_ts_t            rktp_ts_fetch_backoff; /* Back off fetcher for
                                                   * this partition until this
                                                   * absolute timestamp
                                                   * expires. */

	int64_t            rktp_query_offset;    /* Offset to query broker for*/
	int64_t            rktp_next_offset;     /* Next offset to start
                                                  * fetching from.
                                                  * Locality: toppar thread */
	int64_t            rktp_last_next_offset; /* Last next_offset handled
						   * by fetch_decide().
						   * Locality: broker thread */
	int64_t            rktp_app_offset;      /* Last offset delivered to
						  * application + 1 */
	int64_t            rktp_stored_offset;   /* Last stored offset, but
						  * maybe not committed yet. */
        int64_t            rktp_committing_offset; /* Offset currently being
                                                    * committed */
	int64_t            rktp_committed_offset; /* Last committed offset */
	rd_ts_t            rktp_ts_committed_offset; /* Timestamp of last
                                                      * commit */

        struct offset_stats rktp_offsets; /* Current offsets.
                                           * Locality: broker thread*/
        struct offset_stats rktp_offsets_fin; /* Finalized offset for stats.
                                               * Updated periodically
                                               * by broker thread.
                                               * Locks: toppar_lock */

	int64_t rktp_hi_offset;              /* Current high offset.
					      * Locks: toppar_lock */
        int64_t rktp_lo_offset;         
 rd_ts_t            rktp_ts_offset_lag;

	char              *rktp_offset_path;     /* Path to offset file */
	FILE              *rktp_offset_fp;       /* Offset file pointer */
        rd_kafka_cgrp_t   *rktp_cgrp;            /* Belongs to this cgrp */

        int                rktp_assigned;   /* Partition in cgrp assignment */

        rd_kafka_replyq_t  rktp_replyq; /* Current replyq+version
					 * for propagating
					 * major operations, e.g.,
					 * FETCH_STOP. */
	int                rktp_flags;

        shptr_rd_kafka_toppar_t *rktp_s_for_desp; /* Shared pointer for
                                                   * rkt_desp list */
        shptr_rd_kafka_toppar_t *rktp_s_for_cgrp; /* Shared pointer for
                                                   * rkcg_toppars list */
        shptr_rd_kafka_toppar_t *rktp_s_for_rkb;  /* Shared pointer for
                                                   * rkb_toppars list */

	/*
	 * Timers
	 */
	rd_kafka_timer_t rktp_offset_query_tmr;  /* Offset query timer */
	rd_kafka_timer_t rktp_offset_commit_tmr; /* Offset commit timer */
	rd_kafka_timer_t rktp_offset_sync_tmr;   /* Offset file sync timer */
        rd_kafka_timer_t rktp_consumer_lag_tmr;  /* Consumer lag monitoring
						  * timer */

        int rktp_wait_consumer_lag_resp;         /* Waiting for consumer lag
                                                  * response. */

	struct {
		rd_atomic64_t tx_msgs;
		rd_atomic64_t tx_bytes;
                rd_atomic64_t msgs;
                rd_atomic64_t rx_ver_drops;
	} rktp_c;
}
```
* 创建一个 `rd_kafka_toppar_t`对象 `rd_kafka_toppar_new0`:
```
shptr_rd_kafka_toppar_t *rd_kafka_toppar_new0 (rd_kafka_itopic_t *rkt,
					       int32_t partition,
					       const char *func, int line) {
	rd_kafka_toppar_t *rktp;

       // 分配内存
	rktp = rd_calloc(1, sizeof(*rktp));

        // 各项赋值
	rktp->rktp_partition = partition;

        // 属于哪个topic
	rktp->rktp_rkt = rkt;

        rktp->rktp_leader_id = -1;
	rktp->rktp_fetch_state = RD_KAFKA_TOPPAR_FETCH_NONE;
        rktp->rktp_fetch_msg_max_bytes
            = rkt->rkt_rk->rk_conf.fetch_msg_max_bytes;
	rktp->rktp_offset_fp = NULL;
        rd_kafka_offset_stats_reset(&rktp->rktp_offsets);
        rd_kafka_offset_stats_reset(&rktp->rktp_offsets_fin);
        rktp->rktp_hi_offset = RD_KAFKA_OFFSET_INVALID;
	rktp->rktp_lo_offset = RD_KAFKA_OFFSET_INVALID;
	rktp->rktp_app_offset = RD_KAFKA_OFFSET_INVALID;
        rktp->rktp_stored_offset = RD_KAFKA_OFFSET_INVALID;
        rktp->rktp_committed_offset = RD_KAFKA_OFFSET_INVALID;
	rd_kafka_msgq_init(&rktp->rktp_msgq);
        rktp->rktp_msgq_wakeup_fd = -1;
	rd_kafka_msgq_init(&rktp->rktp_xmit_msgq);
	mtx_init(&rktp->rktp_lock, mtx_plain);

        rd_refcnt_init(&rktp->rktp_refcnt, 0);
	rktp->rktp_fetchq = rd_kafka_q_new(rkt->rkt_rk);
        rktp->rktp_ops    = rd_kafka_q_new(rkt->rkt_rk);
        rktp->rktp_ops->rkq_serve = rd_kafka_toppar_op_serve;
        rktp->rktp_ops->rkq_opaque = rktp;
        rd_atomic32_init(&rktp->rktp_version, 1);
	rktp->rktp_op_version = rd_atomic32_get(&rktp->rktp_version);

        // 开始一个timer, 来定时统计消息的lag情况, 目前看是一个`rd_kafka_toppar_t`对象就一个timer, 太多了, 可以用时间轮来作所有partiton的timer
        if (rktp->rktp_rkt->rkt_rk->rk_conf.stats_interval_ms > 0 &&
            rkt->rkt_rk->rk_type == RD_KAFKA_CONSUMER &&
            rktp->rktp_partition != RD_KAFKA_PARTITION_UA) {
                int intvl = rkt->rkt_rk->rk_conf.stats_interval_ms;
                if (intvl < 10 * 1000 /* 10s */)
                        intvl = 10 * 1000;
		rd_kafka_timer_start(&rkt->rkt_rk->rk_timers,
				     &rktp->rktp_consumer_lag_tmr,
                                     intvl * 1000ll,
				     rd_kafka_toppar_consumer_lag_tmr_cb,
				     rktp);
        }

        rktp->rktp_s_rkt = rd_kafka_topic_keep(rkt);

        // 设置其fwd op queue到rd_kakfa_t中的rd_ops, 这样这个rd_kafka_toppar_t对象用到的ops_queue就是rd_kafka_t的了
	rd_kafka_q_fwd_set(rktp->rktp_ops, rkt->rkt_rk->rk_ops);
	rd_kafka_dbg(rkt->rkt_rk, TOPIC, "TOPPARNEW", "NEW %s [%"PRId32"] %p (at %s:%d)",
		     rkt->rkt_topic->str, rktp->rktp_partition, rktp,
		     func, line);

	return rd_kafka_toppar_keep_src(func, line, rktp);
}
```
* 销毁一个`rd_kafka_toppar_t`对象`rd_kafka_toppar_destroy_final `
```
void rd_kafka_toppar_destroy_final (rd_kafka_toppar_t *rktp) {
        // 停掉相应的timer, 清空ops queue
        rd_kafka_toppar_remove(rktp);

        // 将msgq中的kafka message回调给app层后清空
	rd_kafka_dr_msgq(rktp->rktp_rkt, &rktp->rktp_msgq,
			 RD_KAFKA_RESP_ERR__DESTROY);
	rd_kafka_q_destroy_owner(rktp->rktp_fetchq);
        rd_kafka_q_destroy_owner(rktp->rktp_ops);

	rd_kafka_replyq_destroy(&rktp->rktp_replyq);

	rd_kafka_topic_destroy0(rktp->rktp_s_rkt);

	mtx_destroy(&rktp->rktp_lock);

        rd_refcnt_destroy(&rktp->rktp_refcnt);

	rd_free(rktp);
}
```
* 从一个`rd_kafka_itopic_t`(这个我们后面会有专门篇章来介绍, 这里只需要知道它表示topic即可, 里面包括属于它的parition列表)获取指定parition:
```
shptr_rd_kafka_toppar_t *rd_kafka_toppar_get0 (const char *func, int line,
                                               const rd_kafka_itopic_t *rkt,
                                               int32_t partition,
                                               int ua_on_miss) {
        shptr_rd_kafka_toppar_t *s_rktp;
 
        // 数组索引下标来获取 partition
	if (partition >= 0 && partition < rkt->rkt_partition_cnt)
		s_rktp = rkt->rkt_p[partition];
	else if (partition == RD_KAFKA_PARTITION_UA || ua_on_miss)
		s_rktp = rkt->rkt_ua;
	else
		return NULL;

	if (s_rktp)
               // 引用计数加1 
                return rd_kafka_toppar_keep_src(func,line,
                                                rd_kafka_toppar_s2i(s_rktp));

	return NULL;
}
```
* 按topic名字和partition来获取一个`rd_kafka_toppar_t`对象, 没有找到topic, 就先创建这个 ` rd_kafka_itopic_t`对象
```
shptr_rd_kafka_toppar_t *rd_kafka_toppar_get2 (rd_kafka_t *rk,
                                               const char *topic,
                                               int32_t partition,
                                               int ua_on_miss,
                                               int create_on_miss) {
	shptr_rd_kafka_itopic_t *s_rkt;
        rd_kafka_itopic_t *rkt;
        shptr_rd_kafka_toppar_t *s_rktp;

        rd_kafka_wrlock(rk);

        /* Find or create topic */
        // 所有的 rd_kafka_itopic_t对象都存在rd_kafka_t的rkt_topic的tailq队列里, 这里先查找
	if (unlikely(!(s_rkt = rd_kafka_topic_find(rk, topic, 0/*no-lock*/)))) {
                if (!create_on_miss) {
                        rd_kafka_wrunlock(rk);
                        return NULL;
                }
                // 没找到就先创建  rd_kafka_itopic_t对象
                s_rkt = rd_kafka_topic_new0(rk, topic, NULL,
					    NULL, 0/*no-lock*/);
                if (!s_rkt) {
                        rd_kafka_wrunlock(rk);
                        rd_kafka_log(rk, LOG_ERR, "TOPIC",
                                     "Failed to create local topic \"%s\": %s",
                                     topic, rd_strerror(errno));
                        return NULL;
                }
        }

        rd_kafka_wrunlock(rk);

        rkt = rd_kafka_topic_s2i(s_rkt);

	rd_kafka_topic_wrlock(rkt);
	s_rktp = rd_kafka_toppar_desired_add(rkt, partition);
	rd_kafka_topic_wrunlock(rkt);

        rd_kafka_topic_destroy0(s_rkt);

	return s_rktp;
}
```
* `desired partition`: desired partition状态的parititon, 源码中的解释如下:
>  The desired partition list is the list of partitions that are desired
(e.g., by the consumer) but not yet seen on a broker.
As soon as the partition is seen on a broker the toppar is moved from
the desired list and onto the normal rkt_p array.
When the partition on the broker goes away a desired partition is put
back on the desired list

简单说就是需要某一个partition, 但是这个parition的具体信息还没从broker拿掉,这样的parition就是desired parition, 在`rd_kafka_itopic_t`中有一个`rkt_desp`的list, 专门用来存这样的parition, 针对其有如下几个操作,都比较简单:
```
rd_kafka_toppar_desired_get
rd_kafka_toppar_desired_link
rd_kafka_toppar_desired_unlink
rd_kafka_toppar_desired_add0
rd_kafka_toppar_desired_add
rd_kafka_toppar_desired_del
```
* partition在broker间迁移`rd_kafka_toppar_broker_migrate`:
```
static void rd_kafka_toppar_broker_migrate (rd_kafka_toppar_t *rktp,
                                            rd_kafka_broker_t *old_rkb,
                                            rd_kafka_broker_t *new_rkb) {
        rd_kafka_op_t *rko;
        rd_kafka_broker_t *dest_rkb;
        int had_next_leader = rktp->rktp_next_leader ? 1 : 0;

        /* Update next leader */
        if (new_rkb)
                rd_kafka_broker_keep(new_rkb);
        if (rktp->rktp_next_leader)
                rd_kafka_broker_destroy(rktp->rktp_next_leader);
        rktp->rktp_next_leader = new_rkb;
        
        // 在迁移没完成时有可能再次迁移了, 这个时候是不是需要加锁? 
        if (had_next_leader)
                return;

	if (rktp->rktp_fetch_state == RD_KAFKA_TOPPAR_FETCH_OFFSET_WAIT) {
		rd_kafka_toppar_set_fetch_state(
			rktp, RD_KAFKA_TOPPAR_FETCH_OFFSET_QUERY);
		rd_kafka_timer_start(&rktp->rktp_rkt->rkt_rk->rk_timers,
				     &rktp->rktp_offset_query_tmr,
				     500*1000,
				     rd_kafka_offset_query_tmr_cb,
				     rktp);
	}

        //  迁移前broker放到LEAVE op
        if (old_rkb) {
                rko = rd_kafka_op_new(RD_KAFKA_OP_PARTITION_LEAVE);
                dest_rkb = old_rkb;
        } else {
                /* No existing broker, send join op directly to new leader. */
                rko = rd_kafka_op_new(RD_KAFKA_OP_PARTITION_JOIN);
                dest_rkb = new_rkb;
        }

        rko->rko_rktp = rd_kafka_toppar_keep(rktp);

        rd_kafka_q_enq(dest_rkb->rkb_ops, rko);
}
```
* broker的delegate操作:
```
void rd_kafka_toppar_broker_delegate (rd_kafka_toppar_t *rktp,
				      rd_kafka_broker_t *rkb,
				      int for_removal) {
        rd_kafka_t *rk = rktp->rktp_rkt->rkt_rk;
        int internal_fallback = 0;

        /* Delegate toppars with no leader to the
         * internal broker for bookkeeping. */
        // 如果迁移到的broker是NULL, 就获取一个internal broker -> rkb
        if (!rkb && !for_removal && !rd_kafka_terminating(rk)) {
                rkb = rd_kafka_broker_internal(rk);
                internal_fallback = 1;
        }

	if (rktp->rktp_leader == rkb && !rktp->rktp_next_leader) {
                rd_kafka_dbg(rktp->rktp_rkt->rkt_rk, TOPIC, "BRKDELGT",
			     "%.*s [%"PRId32"]: not updating broker: "
                             "already on correct broker %s",
			     RD_KAFKAP_STR_PR(rktp->rktp_rkt->rkt_topic),
			     rktp->rktp_partition,
                             rkb ? rd_kafka_broker_name(rkb) : "(none)");

                if (internal_fallback)
                        rd_kafka_broker_destroy(rkb);
		return;
        }

        // 实际的迁移操作
        if (rktp->rktp_leader || rkb)
                rd_kafka_toppar_broker_migrate(rktp, rktp->rktp_leader, rkb);

        if (internal_fallback)
                rd_kafka_broker_destroy(rkb);
}
```
* 提交offstet到broker `rd_kafka_toppar_offset_commit`
```
void rd_kafka_toppar_offset_commit (rd_kafka_toppar_t *rktp, int64_t offset,
				    const char *metadata) {
        rd_kafka_topic_partition_list_t *offsets;
        rd_kafka_topic_partition_t *rktpar;

        // 构造 一个rd_kafka_topic_partition_list, 把当前的topic添加进去, 包括要提交的offset
        offsets = rd_kafka_topic_partition_list_new(1);
        rktpar = rd_kafka_topic_partition_list_add(
                offsets, rktp->rktp_rkt->rkt_topic->str, rktp->rktp_partition);
        rktpar->offset = offset;
        if (metadata) {
                rktpar->metadata = rd_strdup(metadata);
                rktpar->metadata_size = strlen(metadata);
        }

        // rd_kafka_toppar_t对象更新rktp_committing_offset，表示正在提交的offset
        rktp->rktp_committing_offset = offset;

       // 异步提交offset, 这个操作在之后介绍kafka consumer是会详细分析
        rd_kafka_commit(rktp->rktp_rkt->rkt_rk, offsets, 1/*async*/);

        rd_kafka_topic_partition_list_destroy(offsets);
}
```
* 设置下一次拉取数据时开始的offset位置，即`rd_kafka_toppar_t`的`rktp_next_offset`
```
void rd_kafka_toppar_next_offset_handle (rd_kafka_toppar_t *rktp,
                                         int64_t Offset) {
        // 如果Offset是BEGINNING，END， 发起一个rd_kafka_toppar_offset_request操作，从broker获取offset
        // 如果Offset是RD_KAFKA_OFFSET_INVALID， 需要enqueue一个error op, 设置fetch状态为RD_KAFKA_TOPPAR_FETCH_NONE
        if (RD_KAFKA_OFFSET_IS_LOGICAL(Offset)) {
                /* Offset storage returned logical offset (e.g. "end"),
                 * look it up. */
                rd_kafka_offset_reset(rktp, Offset, RD_KAFKA_RESP_ERR_NO_ERROR,
                                      "update");
                return;
        }

        /* Adjust by TAIL count if, if wanted */
        // 获取从tail开始往前推cnt个offset的位置
        if (rktp->rktp_query_offset <=
            RD_KAFKA_OFFSET_TAIL_BASE) {
                int64_t orig_Offset = Offset;
                int64_t tail_cnt =
                        llabs(rktp->rktp_query_offset -
                              RD_KAFKA_OFFSET_TAIL_BASE);

                if (tail_cnt > Offset)
                        Offset = 0;
                else
                        Offset -= tail_cnt;
        }

        //设置rktp_next_offset
        rktp->rktp_next_offset = Offset;

        rd_kafka_toppar_set_fetch_state(rktp, RD_KAFKA_TOPPAR_FETCH_ACTIVE);

        /* Wake-up broker thread which might be idling on IO */
        if (rktp->rktp_leader)
                rd_kafka_broker_wakeup(rktp->rktp_leader);

}
```
* 从coordinattor获取已提交的offset(FetchOffsetRequest) `rd_kafka_toppar_offset_fetch`:
```
void rd_kafka_toppar_offset_fetch (rd_kafka_toppar_t *rktp,
                                   rd_kafka_replyq_t replyq) {
        rd_kafka_t *rk = rktp->rktp_rkt->rkt_rk;
        rd_kafka_topic_partition_list_t *part;
        rd_kafka_op_t *rko;

        part = rd_kafka_topic_partition_list_new(1);
        rd_kafka_topic_partition_list_add0(part,
                                           rktp->rktp_rkt->rkt_topic->str,
                                           rktp->rktp_partition,
					   rd_kafka_toppar_keep(rktp));

        // 构造OffsetFetch的operator
        rko = rd_kafka_op_new(RD_KAFKA_OP_OFFSET_FETCH);
	rko->rko_rktp = rd_kafka_toppar_keep(rktp);
	rko->rko_replyq = replyq;

	rko->rko_u.offset_fetch.partitions = part;
	rko->rko_u.offset_fetch.do_free = 1;

        // OffsetFetch 请求是与消费有关的，放入cgrp的op queue里
        rd_kafka_q_enq(rktp->rktp_cgrp->rkcg_ops, rko);
}
```
* 获取用于消费的有效的offset
```
void rd_kafka_toppar_offset_request (rd_kafka_toppar_t *rktp,
				     int64_t query_offset, int backoff_ms) {
	rd_kafka_broker_t *rkb;
        rkb = rktp->rktp_leader;

         // 如果rkb是无效的，需要下一个timer来定时query
        if (!backoff_ms && (!rkb || rkb->rkb_source == RD_KAFKA_INTERNAL))
                backoff_ms = 500;

        if (backoff_ms) {
                rd_kafka_toppar_set_fetch_state(
                        rktp, RD_KAFKA_TOPPAR_FETCH_OFFSET_QUERY);
                // 启动timer, timer到期会执行rd_kafka_offset_query_tmr_cb回调，这个回调还是调用当前这个函数
		rd_kafka_timer_start(&rktp->rktp_rkt->rkt_rk->rk_timers,
				     &rktp->rktp_offset_query_tmr,
				     backoff_ms*1000ll,
				     rd_kafka_offset_query_tmr_cb, rktp);
		return;
        }

        // stop这个重试的timer
        rd_kafka_timer_stop(&rktp->rktp_rkt->rkt_rk->rk_timers,
                            &rktp->rktp_offset_query_tmr, 1/*lock*/);

        // 从coordinattor获取需要消费的offset
	if (query_offset == RD_KAFKA_OFFSET_STORED &&
            rktp->rktp_rkt->rkt_conf.offset_store_method ==
            RD_KAFKA_OFFSET_METHOD_BROKER) {
                /*
                 * Get stored offset from broker based storage:
                 * ask cgrp manager for offsets
                 */
                rd_kafka_toppar_offset_fetch(
			rktp,
			RD_KAFKA_REPLYQ(rktp->rktp_ops,
					rktp->rktp_op_version));

	} else {
                shptr_rd_kafka_toppar_t *s_rktp;
                rd_kafka_topic_partition_list_t *offsets;

                /*
                 * Look up logical offset (end,beginning,tail,..)
                 */
                s_rktp = rd_kafka_toppar_keep(rktp);

		if (query_offset <= RD_KAFKA_OFFSET_TAIL_BASE)
			query_offset = RD_KAFKA_OFFSET_END;

                offsets = rd_kafka_topic_partition_list_new(1);
                rd_kafka_topic_partition_list_add(
                        offsets,
                        rktp->rktp_rkt->rkt_topic->str,
                        rktp->rktp_partition)->offset = query_offset;
                
                // 基本上用于reset offset, 获取当前partition的最旧offset或最新offset
                rd_kafka_OffsetRequest(rkb, offsets, 0,
                                       RD_KAFKA_REPLYQ(rktp->rktp_ops,
                                                       rktp->rktp_op_version),
                                       rd_kafka_toppar_handle_Offset,
                                       s_rktp);

                rd_kafka_topic_partition_list_destroy(offsets);
        }

        rd_kafka_toppar_set_fetch_state(rktp,
				RD_KAFKA_TOPPAR_FETCH_OFFSET_WAIT);
}
```

---
### [Librdkafka源码分析-Content Table](https://www.jianshu.com/p/1a94bb09a6e6)