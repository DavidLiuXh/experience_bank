* 我们在之前的[Kafka源码分析系列](https://www.jianshu.com/p/aa274f8fe00f)中介绍过[kafka集群的metadata](https://www.jianshu.com/p/b8e7996ea9af), 大家可以参考一下;

* 简单说, kafka集群的metadata包括:
   1. 所有broker的信息: ip和port;
   2. 所有topic的信息: topic name, partition数量, 每个partition的leader, isr, replica集合等
* kafka集群的每一台broker都缓存了整个集群的metadata, 当broker或某一个topic的metadata信息发生变化时, 集群的[Controller](https://www.jianshu.com/p/a9f559ee6035) 都会感知到作相应的状态转换, 同时把发生变化的新的metadata信息广播到所有的broker;
* 下面我们介绍一下librdkafka中对metadata的封装和操作,基本上就是metadata的获取,定时刷新以及引用的操作, 比如说partition leader的迁移, partition个数的变化, broker的上下线等等;
--- 
###### Metadata的获取
* 我们先来看一下metadata的定义, 在`rdkafka.h`中
 1. kafka集群整体metadata的定义, 包括broker, topic, partition三部分:
```
typedef struct rd_kafka_metadata {
        int         broker_cnt;     /**< Number of brokers in \p brokers */
        struct rd_kafka_metadata_broker *brokers;  /**< Brokers */

        int         topic_cnt;      /**< Number of topics in \p topics */
        struct rd_kafka_metadata_topic *topics;    /**< Topics */

        int32_t     orig_broker_id;   /**< Broker originating this metadata */ 表示这个metadata是从哪个broker返回的
        char       *orig_broker_name; /**< Name of originating broker */
} rd_kafka_metadata_t;
```
   2. broker的metadata:
```
typedef struct rd_kafka_metadata_broker {
        int32_t     id;             /**< Broker Id */
        char       *host;           /**< Broker hostname */
        int         port;           /**< Broker listening port */
} rd_kafka_metadata_broker_t;
```
  3. topic的metadata:
```
typedef struct rd_kafka_metadata_topic {
        char       *topic;          /**< Topic name */
        int         partition_cnt;  /**< Number of partitions in \p partitions*/
        struct rd_kafka_metadata_partition *partitions; /**< Partitions */
        rd_kafka_resp_err_t err;    /**< Topic error reported by broker */
} rd_kafka_metadata_topic_t;
```
 4. partition的metadata:
```
typedef struct rd_kafka_metadata_partition {
        int32_t     id;             /**< Partition Id */
        rd_kafka_resp_err_t err;    /**< Partition error reported by broker */
        int32_t     leader;         /**< Leader broker */
        int         replica_cnt;    /**< Number of brokers in \p replicas */
        int32_t    *replicas;       /**< Replica brokers */
        int         isr_cnt;        /**< Number of ISR brokers in \p isrs */
        int32_t    *isrs;           /**< In-Sync-Replica brokers */
} rd_kafka_metadata_partition_t;
```
* metadata的获取, 阻塞操作 `rd_kafka_metadata`:
```
rd_kafka_metadata (rd_kafka_t *rk, int all_topics,
                   rd_kafka_topic_t *only_rkt,
                   const struct rd_kafka_metadata **metadatap,
                   int timeout_ms) {
        rd_kafka_q_t *rkq;
        rd_kafka_broker_t *rkb;
        rd_kafka_op_t *rko;
	rd_ts_t ts_end = rd_timeout_init(timeout_ms);
        rd_list_t topics;

        // 在timeout_ms时长的超时时间内选择一台有效的broker,
	rkb = rd_kafka_broker_any_usable(rk, timeout_ms, 1);
	if (!rkb)
		return RD_KAFKA_RESP_ERR__TRANSPORT;

       // 创建一个queue, 作为metadata的response返回完生成的op的replay队列
        rkq = rd_kafka_q_new(rk);

        rd_list_init(&topics, 0, rd_free);
        if (!all_topics) {
                if (only_rkt)
                        rd_list_add(&topics,
                                    rd_strdup(rd_kafka_topic_a2i(only_rkt)->
                                              rkt_topic->str));
                else
                        rd_kafka_local_topics_to_list(rkb->rkb_rk, &topics);
        }

        // 创建op, 里面会关联metadata request的buffer
        rko = rd_kafka_op_new(RD_KAFKA_OP_METADATA);
        rd_kafka_op_set_replyq(rko, rkq, 0);
        rko->rko_u.metadata.force = 1; /* Force metadata request regardless
                                        * of outstanding metadata requests. */
        // 构造metadata request的二进制协议并最终放到broker的发送buffer队列里
        rd_kafka_MetadataRequest(rkb, &topics, "application requested", rko);

        rd_list_destroy(&topics);
        rd_kafka_broker_destroy(rkb);

       // 等待metadata response返回或超时, 如果正常返回,在放入这个replay queue之前, 返回的response二进制协议会被parse到` rd_kafka_metadata`对象中
        rko = rd_kafka_q_pop(rkq, rd_timeout_remains(ts_end), 0);

        rd_kafka_q_destroy_owner(rkq);

        /* Timeout */
        if (!rko)
                return RD_KAFKA_RESP_ERR__TIMED_OUT;

        /* Error */
        if (rko->rko_err) {
                rd_kafka_resp_err_t err = rko->rko_err;
                rd_kafka_op_destroy(rko);
                return err;
        }

        /* Reply: pass metadata pointer to application who now owns it*/
        rd_kafka_assert(rk, rko->rko_u.metadata.md);
        *metadatap = rko->rko_u.metadata.md;
        rko->rko_u.metadata.md = NULL;
        rd_kafka_op_destroy(rko);

        return RD_KAFKA_RESP_ERR_NO_ERROR;
}
```
* metadata的拷贝操作 `rd_kafka_metadata_copy (const struct rd_kafka_metadata *src, size_t size)`:
执行一个深度copy, 使用[rd_tmpabuf_t](https://www.jianshu.com/p/30cd98f04891),保证内存的对齐;

* 处理metadata的response `rd_kafka_parse_Metadata`:
```
rd_kafka_parse_Metadata (rd_kafka_broker_t *rkb,
                         rd_kafka_buf_t *request,
                         rd_kafka_buf_t *rkbuf) {
        rd_kafka_t *rk = rkb->rkb_rk;
        int i, j, k;
        rd_tmpabuf_t tbuf;
        struct rd_kafka_metadata *md;
        size_t rkb_namelen;
        const int log_decode_errors = LOG_ERR;
        rd_list_t *missing_topics = NULL;
        const rd_list_t *requested_topics = request->rkbuf_u.Metadata.topics;
        int all_topics = request->rkbuf_u.Metadata.all_topics;
        const char *reason = request->rkbuf_u.Metadata.reason ?
                request->rkbuf_u.Metadata.reason : "(no reason)";
        int ApiVersion = request->rkbuf_reqhdr.ApiVersion;
        rd_kafkap_str_t cluster_id = RD_ZERO_INIT;
        int32_t controller_id = -1;

        rd_kafka_assert(NULL, thrd_is_current(rk->rk_thread));

        /* Remove topics from missing_topics as they are seen in Metadata. */
        if (requested_topics)
                missing_topics = rd_list_copy(requested_topics,
                                              rd_list_string_copy, NULL);

        rd_kafka_broker_lock(rkb);
        rkb_namelen = strlen(rkb->rkb_name)+1;
        
        // 使用tmpbuf从parse的内存分配
        rd_tmpabuf_new(&tbuf,
                       sizeof(*md) + rkb_namelen + (rkbuf->rkbuf_totlen * 4),
                       0/*dont assert on fail*/);

        if (!(md = rd_tmpabuf_alloc(&tbuf, sizeof(*md))))
                goto err;
        md->orig_broker_id = rkb->rkb_nodeid;
        md->orig_broker_name = rd_tmpabuf_write(&tbuf,
                                                rkb->rkb_name, rkb_namelen);
        rd_kafka_broker_unlock(rkb);

        // 解析并添充broker信息
        .
        .
        .

       // 解析并添充topic和partition信息
       .
       .
       .

        // 如果正在shutdown, 我们就不更新medata信息了
        if (rd_kafka_terminating(rkb->rkb_rk))
                goto done;

        // 如果解析出来的broker个数是0, 或者topic个数是0, 就准备retry吧
        if (md->broker_cnt == 0 && md->topic_cnt == 0) {
                rd_rkb_dbg(rkb, METADATA, "METADATA",
                           "No brokers or topics in metadata: retrying");
                goto err;
        }

        // 更新broker 列表, 具体的更新操作, 我们在分析broker时再具体介绍
       // 已存在的更新,没有的新添加, 最终会开启一个新的broker的io event loop
        for (i = 0 ; i < md->broker_cnt ; i++) {
                rd_kafka_broker_update(rkb->rkb_rk, rkb->rkb_proto,
                                       &md->brokers[i]);
        }

        /* Update partition count and leader for each topic we know about */
        for (i = 0 ; i < md->topic_cnt ; i++) {
                rd_kafka_metadata_topic_t *mdt = &md->topics[i];

                /* Ignore topics in blacklist */
                if (rkb->rkb_rk->rk_conf.topic_blacklist &&
                    rd_kafka_pattern_match(rkb->rkb_rk->rk_conf.topic_blacklist,
                                           mdt->topic)) {
                        rd_rkb_dbg(rkb, TOPIC, "BLACKLIST",
                                   "Ignoring blacklisted topic \"%s\" "
                                   "in metadata", mdt->topic);
                        continue;
                }

                /* Ignore metadata completely for temporary errors. (issue #513)
                 *   LEADER_NOT_AVAILABLE: Broker is rebalancing
                 */
                if (mdt->err == RD_KAFKA_RESP_ERR_LEADER_NOT_AVAILABLE &&
                    mdt->partition_cnt == 0) {
                        rd_list_free_cb(missing_topics,
                                        rd_list_remove_cmp(missing_topics,
                                                           mdt->topic,
                                                           (void *)strcmp));
                        continue;
                }


                // 更新topic detadata, 会涉及到leader的涉及, 处理partition的新增和减少,  都是通过op作的异步操作
                rd_kafka_topic_metadata_update2(rkb, mdt);

                if (requested_topics) {
                        rd_list_free_cb(missing_topics,
                                        rd_list_remove_cmp(missing_topics,
                                                           mdt->topic,
                                                           (void*)strcmp));
                        if (!all_topics) {
                                rd_kafka_wrlock(rk);
                                rd_kafka_metadata_cache_topic_update(rk, mdt);
                                rd_kafka_wrunlock(rk);
                        }
                }
        }


       // 对没有获取到metadata的topic, 作清理操作
        if (missing_topics) {
                char *topic;
                RD_LIST_FOREACH(topic, missing_topics, i) {
                        shptr_rd_kafka_itopic_t *s_rkt;

                        s_rkt = rd_kafka_topic_find(rkb->rkb_rk, topic, 1/*lock*/);
                        if (s_rkt) {
                                rd_kafka_topic_metadata_none(
                                        rd_kafka_topic_s2i(s_rkt));
                                rd_kafka_topic_destroy0(s_rkt);
                        }
                }
        }


        rd_kafka_wrlock(rkb->rkb_rk);
        rkb->rkb_rk->rk_ts_metadata = rd_clock();

        /* Update cached cluster id. */
        if (RD_KAFKAP_STR_LEN(&cluster_id) > 0 &&
            (!rkb->rkb_rk->rk_clusterid ||
             rd_kafkap_str_cmp_str(&cluster_id, rkb->rkb_rk->rk_clusterid))) {
                if (rkb->rkb_rk->rk_clusterid)
                        rd_free(rkb->rkb_rk->rk_clusterid);
                rkb->rkb_rk->rk_clusterid = RD_KAFKAP_STR_DUP(&cluster_id);
        }

        if (all_topics) {
               // 更新 metadata cache
                rd_kafka_metadata_cache_update(rkb->rkb_rk,
                                               md, 1/*abs update*/);

                if (rkb->rkb_rk->rk_full_metadata)
                        rd_kafka_metadata_destroy(rkb->rkb_rk->rk_full_metadata);
                rkb->rkb_rk->rk_full_metadata =
                        rd_kafka_metadata_copy(md, tbuf.of);
                rkb->rkb_rk->rk_ts_full_metadata = rkb->rkb_rk->rk_ts_metadata;
                rd_rkb_dbg(rkb, METADATA, "METADATA",
                           "Caching full metadata with "
                           "%d broker(s) and %d topic(s): %s",
                           md->broker_cnt, md->topic_cnt, reason);
        } else {
                rd_kafka_metadata_cache_expiry_start(rk);
        }

        /* Remove cache hints for the originally requested topics. */
        if (requested_topics)
                rd_kafka_metadata_cache_purge_hints(rk, requested_topics);

        rd_kafka_wrunlock(rkb->rkb_rk);

        /* Check if cgrp effective subscription is affected by
         * new metadata. */
        if (rkb->rkb_rk->rk_cgrp)
                rd_kafka_cgrp_metadata_update_check(
                        rkb->rkb_rk->rk_cgrp, 1/*do join*/);
done:
        if (missing_topics)
                rd_list_destroy(missing_topics);
        return md;

 err_parse:
err:
        if (requested_topics) {
                /* Failed requests shall purge cache hints for
                 * the requested topics. */
                rd_kafka_wrlock(rkb->rkb_rk);
                rd_kafka_metadata_cache_purge_hints(rk, requested_topics);
                rd_kafka_wrunlock(rkb->rkb_rk);
        }

        if (missing_topics)
                rd_list_destroy(missing_topics);

        rd_tmpabuf_destroy(&tbuf);
        return NULL;
}
```
* 刷新给定的topic list的metadata `rd_kafka_metadata_refresh_topics`:
```
rd_kafka_metadata_refresh_topics (rd_kafka_t *rk, rd_kafka_broker_t *rkb,
                                  const rd_list_t *topics, int force,
                                  const char *reason) {
        rd_list_t q_topics;
        int destroy_rkb = 0;

        if (!rk)
                rk = rkb->rkb_rk;

        rd_kafka_wrlock(rk);

        // 确定一下发送metadata request的broker
        if (!rkb) {
                if (!(rkb = rd_kafka_broker_any_usable(rk, RD_POLL_NOWAIT, 0))){
                        rd_kafka_wrunlock(rk);
                        rd_kafka_dbg(rk, METADATA, "METADATA",
                                     "Skipping metadata refresh of %d topic(s):"
                                     " no usable brokers",
                                     rd_list_cnt(topics));
                        return RD_KAFKA_RESP_ERR__TRANSPORT;
                }
                destroy_rkb = 1;
        }

        rd_list_init(&q_topics, rd_list_cnt(topics), rd_free);

        if (!force) {
                // 如查不是强制必须立刻刷新的话, 只是把这个topic对应在metadata cache中的状态改为RD_KAFKA_RESP_ERR__WAIT_CACHE, 设置新的刷新时间
                rd_kafka_metadata_cache_hint(rk, topics, &q_topics,
                                             0/*dont replace*/);
                rd_kafka_wrunlock(rk);

                if (rd_list_cnt(&q_topics) == 0) {
                        rd_list_destroy(&q_topics);
                        if (destroy_rkb)
                                rd_kafka_broker_destroy(rkb);
                        return RD_KAFKA_RESP_ERR_NO_ERROR;
                }

        } else {
                // 立即发送metadata request
                rd_kafka_wrunlock(rk);
                rd_list_copy_to(&q_topics, topics, rd_list_string_copy, NULL);
        }

        rd_kafka_MetadataRequest(rkb, &q_topics, reason, NULL);

        rd_list_destroy(&q_topics);

        if (destroy_rkb)
                rd_kafka_broker_destroy(rkb);

        return RD_KAFKA_RESP_ERR_NO_ERROR;
}
```
* 只请求broker相关的metadata
```
rd_kafka_resp_err_t
rd_kafka_metadata_refresh_brokers (rd_kafka_t *rk, rd_kafka_broker_t *rkb,
                                   const char *reason) {
        return rd_kafka_metadata_request(rk, rkb, NULL /*brokers only*/,
                                         reason, NULL);
}
```
* 一个快速地刷新partition leader的操作 `rd_kafka_metadata_fast_leader_query`
这个`rkmc_query_tmr`是专门真对没有leader的topic的定期刷新
```
void rd_kafka_metadata_fast_leader_query (rd_kafka_t *rk) {
        rd_ts_t next;

        /* Restart the timer if it will speed things up. */
        next = rd_kafka_timer_next(&rk->rk_timers,
                                   &rk->rk_metadata_cache.rkmc_query_tmr,
                                   1/*lock*/);
        if (next == -1 /* not started */ ||
            next > rk->rk_conf.metadata_refresh_fast_interval_ms*1000) {
                //开始一个fast ledader query定时期, 过期执行 rd_kafka_metadata_leader_query_tmr_cb
                rd_kafka_timer_start(&rk->rk_timers,
                                     &rk->rk_metadata_cache.rkmc_query_tmr,
                                     rk->rk_conf.
                                     metadata_refresh_fast_interval_ms*1000,
                                     rd_kafka_metadata_leader_query_tmr_cb,
                                     NULL);
        }
}
```
* leader定时刷新处理回调函数 `rd_kafka_metadata_leader_query_tmr_cb`:
```
static void rd_kafka_metadata_leader_query_tmr_cb (rd_kafka_timers_t *rkts,
                                                   void *arg) {
        rd_kafka_t *rk = rkts->rkts_rk;
        rd_kafka_timer_t *rtmr = &rk->rk_metadata_cache.rkmc_query_tmr;
        rd_kafka_itopic_t *rkt;
        rd_list_t topics;

        rd_kafka_wrlock(rk);
        rd_list_init(&topics, rk->rk_topic_cnt, rd_free);

        TAILQ_FOREACH(rkt, &rk->rk_topics, rkt_link) {
                int i, no_leader = 0;
                rd_kafka_topic_rdlock(rkt);

                //  跳过不存在的topic的处理
                if (rkt->rkt_state == RD_KAFKA_TOPIC_S_NOTEXISTS) {
                        /* Skip topics that are known to not exist. */
                        rd_kafka_topic_rdunlock(rkt);
                        continue;
                }

                no_leader = rkt->rkt_flags & RD_KAFKA_TOPIC_F_LEADER_UNAVAIL;

               // 如果某个topic中有一个partition没有leader就是个没有leader的topic
                for (i = 0 ; !no_leader && i < rkt->rkt_partition_cnt ; i++) {
                        rd_kafka_toppar_t *rktp =
                                rd_kafka_toppar_s2i(rkt->rkt_p[i]);
                        rd_kafka_toppar_lock(rktp);
                        no_leader = !rktp->rktp_leader &&
                                !rktp->rktp_next_leader;
                        rd_kafka_toppar_unlock(rktp);
                }

                if (no_leader || rkt->rkt_partition_cnt == 0)
                        rd_list_add(&topics, rd_strdup(rkt->rkt_topic->str));

                rd_kafka_topic_rdunlock(rkt);
        }

        rd_kafka_wrunlock(rk);

        if (rd_list_cnt(&topics) == 0) {
                /* No leader-less topics+partitions, stop the timer. */
                rd_kafka_timer_stop(rkts, rtmr, 1/*lock*/);
        } else {
                // 针对选择好的topic发送metadata request
                rd_kafka_metadata_refresh_topics(rk, NULL, &topics, 1/*force*/,
                                                 "partition leader query");
                /* Back off next query exponentially until we reach
                 * the standard query interval - then stop the timer
                 * since the intervalled querier will do the job for us. */
                if (rk->rk_conf.metadata_refresh_interval_ms > 0 &&
                    rtmr->rtmr_interval * 2 / 1000 >=
                    rk->rk_conf.metadata_refresh_interval_ms)
                        rd_kafka_timer_stop(rkts, rtmr, 1/*lock*/);
                else
                        rd_kafka_timer_backoff(rkts, rtmr,
                                               (int)rtmr->rtmr_interval);
        }

        rd_list_destroy(&topics);
}
```
###### Metadata在内存中的cache及其操作
* cache相关结构体
 1. cache entry:
```
struct rd_kafka_metadata_cache_entry {
        rd_avl_node_t rkmce_avlnode;             /* rkmc_avl */
        TAILQ_ENTRY(rd_kafka_metadata_cache_entry) rkmce_link; /* rkmc_expiry */
        rd_ts_t rkmce_ts_expires;                /* Expire time */ 这个entry对应的metadata下次刷新的时间
        rd_ts_t rkmce_ts_insert;                 /* Insert time */
        rd_kafka_metadata_topic_t rkmce_mtopic;  /* Cached topic metadata */  对应的metadata信息
        /* rkmce_partitions memory points here. */
};
```
  2. cache结构体定义:
```
struct rd_kafka_metadata_cache {
        rd_avl_t         rkmc_avl; // 使用红黑树来存储每个entry, key为topic name, 加速查找
        TAILQ_HEAD(, rd_kafka_metadata_cache_entry) rkmc_expiry; // 使用tailq来存储所有被cached的entry, 过期时间早的会被排在 tailq的前面
        rd_kafka_timer_t rkmc_expiry_tmr;
        int              rkmc_cnt;

        /* Protected by full_lock: */
       // 针对所有topic或broker的metadata的请求在repsonse没有返回之前最多只能发送一个
        mtx_t            rkmc_full_lock;
        int              rkmc_full_topics_sent; /* Full MetadataRequest for
                                                 * all topics has been sent,
                                                 * awaiting response. */
        int              rkmc_full_brokers_sent; /* Full MetadataRequest for
                                                  * all brokers (but not topics)
                                                  * has been sent,
                                                  * awaiting response. */
        // 用于没有leader的topic的定时metadata请求
        rd_kafka_timer_t rkmc_query_tmr; /* Query timer for topic's without
                                          * leaders. */
        cnd_t            rkmc_cnd;       /* cache_wait_change() cond. */
        mtx_t            rkmc_cnd_lock;  /* lock for rkmc_cnd */
};
```
* cache的初始化 `rd_kafka_metadata_cache_init`:
```
void rd_kafka_metadata_cache_init (rd_kafka_t *rk) {
        // 初始化红黑树
        rd_avl_init(&rk->rk_metadata_cache.rkmc_avl,
                    rd_kafka_metadata_cache_entry_cmp, 0);
       // 初初化tailq 
        TAILQ_INIT(&rk->rk_metadata_cache.rkmc_expiry);
        mtx_init(&rk->rk_metadata_cache.rkmc_full_lock, mtx_plain);
        mtx_init(&rk->rk_metadata_cache.rkmc_cnd_lock, mtx_plain);
        cnd_init(&rk->rk_metadata_cache.rkmc_cnd);

}
```
* cache删除操作 `rd_kafka_metadata_cache_delete`
```
rd_kafka_metadata_cache_delete (rd_kafka_t *rk,
                                struct rd_kafka_metadata_cache_entry *rkmce,
                                int unlink_avl) {
       // 需要从红黑树上摘掉的话就摘掉
        if (unlink_avl)
                RD_AVL_REMOVE_ELM(&rk->rk_metadata_cache.rkmc_avl, rkmce);
        TAILQ_REMOVE(&rk->rk_metadata_cache.rkmc_expiry, rkmce, rkmce_link);
        rd_kafka_assert(NULL, rk->rk_metadata_cache.rkmc_cnt > 0);
        rk->rk_metadata_cache.rkmc_cnt--;

        rd_free(rkmce);
}
```
* cache的查找操作 `rd_kafka_metadata_cache_find`:
从红黑树上查找metadata
```
struct rd_kafka_metadata_cache_entry *
rd_kafka_metadata_cache_find (rd_kafka_t *rk, const char *topic, int valid) {
        struct rd_kafka_metadata_cache_entry skel, *rkmce;
        skel.rkmce_mtopic.topic = (char *)topic;
        rkmce = RD_AVL_FIND(&rk->rk_metadata_cache.rkmc_avl, &skel);
        if (rkmce && (!valid || RD_KAFKA_METADATA_CACHE_VALID(rkmce)))
                return rkmce;
        return NULL;
}
```
* cache的插入操作 `rd_kafka_metadata_cache_insert`
```
static struct rd_kafka_metadata_cache_entry *
rd_kafka_metadata_cache_insert (rd_kafka_t *rk,
                                const rd_kafka_metadata_topic_t *mtopic,
                                rd_ts_t now, rd_ts_t ts_expires) {
        struct rd_kafka_metadata_cache_entry *rkmce, *old;
        size_t topic_len;
        rd_tmpabuf_t tbuf;
        int i;
        
        // 叙事我用tmpbuffer来得新构造一个metadata entry
        topic_len = strlen(mtopic->topic) + 1;
        rd_tmpabuf_new(&tbuf,
                       RD_ROUNDUP(sizeof(*rkmce), 8) +
                       RD_ROUNDUP(topic_len, 8) +
                       (mtopic->partition_cnt *
                        RD_ROUNDUP(sizeof(*mtopic->partitions), 8)),
                       1/*assert on fail*/);

        rkmce = rd_tmpabuf_alloc(&tbuf, sizeof(*rkmce));

        rkmce->rkmce_mtopic = *mtopic;

        /* Copy topic name and update pointer */
        rkmce->rkmce_mtopic.topic = rd_tmpabuf_write_str(&tbuf, mtopic->topic);

        /* Copy partition array and update pointer */
        rkmce->rkmce_mtopic.partitions =
                rd_tmpabuf_write(&tbuf, mtopic->partitions,
                                 mtopic->partition_cnt *
                                 sizeof(*mtopic->partitions));

        /* Clear uncached fields. */
        for (i = 0 ; i < mtopic->partition_cnt ; i++) {
                rkmce->rkmce_mtopic.partitions[i].replicas = NULL;
                rkmce->rkmce_mtopic.partitions[i].replica_cnt = 0;
                rkmce->rkmce_mtopic.partitions[i].isrs = NULL;
                rkmce->rkmce_mtopic.partitions[i].isr_cnt = 0;
        }

        /* Sort partitions for future bsearch() lookups. */
        qsort(rkmce->rkmce_mtopic.partitions,
              rkmce->rkmce_mtopic.partition_cnt,
              sizeof(*rkmce->rkmce_mtopic.partitions),
              rd_kafka_metadata_partition_id_cmp);

        // 插到缓存tailq的队尾
        TAILQ_INSERT_TAIL(&rk->rk_metadata_cache.rkmc_expiry,
                          rkmce, rkmce_link);
        rk->rk_metadata_cache.rkmc_cnt++;
        rkmce->rkmce_ts_expires = ts_expires;
        rkmce->rkmce_ts_insert = now;

        /* Insert (and replace existing) entry. */
        // 插入到红黑树中, 如果已存在,就替换到原有的
        old = RD_AVL_INSERT(&rk->rk_metadata_cache.rkmc_avl, rkmce,
                            rkmce_avlnode);
        if (old)
                rd_kafka_metadata_cache_delete(rk, old, 0);

        /* Explicitly not freeing the tmpabuf since rkmce points to its
         * memory. */
        return rkmce;
}
```
* 清空所有的cache
```
static void rd_kafka_metadata_cache_purge (rd_kafka_t *rk) {
        struct rd_kafka_metadata_cache_entry *rkmce;
        int was_empty = TAILQ_EMPTY(&rk->rk_metadata_cache.rkmc_expiry);
       
        // 清除每一个entry
        while ((rkmce = TAILQ_FIRST(&rk->rk_metadata_cache.rkmc_expiry)))
                rd_kafka_metadata_cache_delete(rk, rkmce, 1);
        // 停掉过期刷新的timter
        rd_kafka_timer_stop(&rk->rk_timers,
                            &rk->rk_metadata_cache.rkmc_expiry_tmr, 1);

        // brordcast 状态 
        if (!was_empty)
                rd_kafka_metadata_cache_propagate_changes(rk);
}
```
* cache的更新 `rd_kafka_metadata_cache_topic_update`
```
rd_kafka_metadata_cache_topic_update (rd_kafka_t *rk,
                                      const rd_kafka_metadata_topic_t *mdt) {
        rd_ts_t now = rd_clock();
        rd_ts_t ts_expires = now + (rk->rk_conf.metadata_max_age_ms * 1000);
        int changed = 1;

        if (!mdt->err)
                // 获取的metadata没有错误,就insert
                rd_kafka_metadata_cache_insert(rk, mdt, now, ts_expires);
        else
                // 获取的metadata有错误, 有删除
                changed = rd_kafka_metadata_cache_delete_by_name(rk,
                                                                 mdt->topic);
        // 插入了新的,或者成功删除了已有的, 都表明cache有改变, 需要broadcast状态
        if (changed)
                rd_kafka_metadata_cache_propagate_changes(rk);
}
```
* 开始或更新过期刷新metadata的timer
如果调用时这个timer已经开始就更新下它的expire时间
```
void rd_kafka_metadata_cache_expiry_start (rd_kafka_t *rk) {
        struct rd_kafka_metadata_cache_entry *rkmce;

        if ((rkmce = TAILQ_FIRST(&rk->rk_metadata_cache.rkmc_expiry)))
                rd_kafka_timer_start(&rk->rk_timers,
                                     &rk->rk_metadata_cache.rkmc_expiry_tmr,
                                     rkmce->rkmce_ts_expires - rd_clock(),
                                     rd_kafka_metadata_cache_evict_tmr_cb,
                                     rk);
}
```
* 过期刷新时的回调函数
```
static int rd_kafka_metadata_cache_evict (rd_kafka_t *rk) {
        int cnt = 0;
        rd_ts_t now = rd_clock();
        struct rd_kafka_metadata_cache_entry *rkmce;

        //过期时间早的会被排在 tailq的前面, 删除所有过期的cache
        while ((rkmce = TAILQ_FIRST(&rk->rk_metadata_cache.rkmc_expiry)) &&
               rkmce->rkmce_ts_expires <= now) {
                rd_kafka_metadata_cache_delete(rk, rkmce, 1);
                cnt++;
        }

        if (rkmce)
                // 删除所有过期的cacher后,若队列不为空, 则开始新的timer
                rd_kafka_timer_start(&rk->rk_timers,
                                     &rk->rk_metadata_cache.rkmc_expiry_tmr,
                                     rkmce->rkmce_ts_expires - now,
                                     rd_kafka_metadata_cache_evict_tmr_cb,
                                     rk);
        else
                // // 删除所有过期的cacher后,若队列为空, 则stop timer
                rd_kafka_timer_stop(&rk->rk_timers,
                                    &rk->rk_metadata_cache.rkmc_expiry_tmr, 1);

        if (cnt)
                rd_kafka_metadata_cache_propagate_changes(rk);

        return cnt;
}
```
* 根据topic名字获取对应的topic metadata
```
const rd_kafka_metadata_topic_t *
rd_kafka_metadata_cache_topic_get (rd_kafka_t *rk, const char *topic,
                                   int valid) {
        struct rd_kafka_metadata_cache_entry *rkmce;

        if (!(rkmce = rd_kafka_metadata_cache_find(rk, topic, valid)))
                return NULL;

        return &rkmce->rkmce_mtopic;
}
```
* 根据topic-parition获取对应的partition metadata
```
int rd_kafka_metadata_cache_topic_partition_get (
        rd_kafka_t *rk,
        const rd_kafka_metadata_topic_t **mtopicp,
        const rd_kafka_metadata_partition_t **mpartp,
        const char *topic, int32_t partition, int valid) {

        const rd_kafka_metadata_topic_t *mtopic;
        const rd_kafka_metadata_partition_t *mpart;
        rd_kafka_metadata_partition_t skel = { .id = partition };

        *mtopicp = NULL;
        *mpartp = NULL;

        if (!(mtopic = rd_kafka_metadata_cache_topic_get(rk, topic, valid)))
                return -1;

        *mtopicp = mtopic;

        /* Partitions array may be sparse so use bsearch lookup. */
       // 二分法查找
        mpart = bsearch(&skel, mtopic->partitions,
                        mtopic->partition_cnt,
                        sizeof(*mtopic->partitions),
                        rd_kafka_metadata_partition_id_cmp);

        if (!mpart)
                return 0;

        *mpartp = mpart;

        return 1;
}
```
* 广播cache内容的变化
```
static void rd_kafka_metadata_cache_propagate_changes (rd_kafka_t *rk) {
        mtx_lock(&rk->rk_metadata_cache.rkmc_cnd_lock);
        cnd_broadcast(&rk->rk_metadata_cache.rkmc_cnd);
        mtx_unlock(&rk->rk_metadata_cache.rkmc_cnd_lock);
}
* 等待cache内容的变化
```
int rd_kafka_metadata_cache_wait_change (rd_kafka_t *rk, int timeout_ms) {
        int r;

        mtx_lock(&rk->rk_metadata_cache.rkmc_cnd_lock);
        r = cnd_timedwait_ms(&rk->rk_metadata_cache.rkmc_cnd,
                             &rk->rk_metadata_cache.rkmc_cnd_lock,
                             timeout_ms);
        mtx_unlock(&rk->rk_metadata_cache.rkmc_cnd_lock);

        return r == thrd_success;
}
```
* 清除给定的topic列表里对应的cache,在红黑树中找不到的,或者找到了但状态是RD_KAFKA_RESP_ERR__WAIT_CACHE的都不需要删除
```
void rd_kafka_metadata_cache_purge_hints (rd_kafka_t *rk,
                                          const rd_list_t *topics) {
        const char *topic;
        int i;
        int cnt = 0;

        RD_LIST_FOREACH(topic, topics, i) {
                struct rd_kafka_metadata_cache_entry *rkmce;
                // 在红黑树中找不到的,或者找到了但状态是RD_KAFKA_RESP_ERR__WAIT_CACHE的都不需要删除
                if (!(rkmce = rd_kafka_metadata_cache_find(rk, topic,
                                                           0/*any*/)) ||
                    RD_KAFKA_METADATA_CACHE_VALID(rkmce))
                        continue;

                rd_kafka_metadata_cache_delete(rk, rkmce, 1/*unlink avl*/);
                cnt++;
        }

        if (cnt > 0) {
                rd_kafka_metadata_cache_propagate_changes(rk);
        }
}
```
---
### [Librdkafka源码分析-Content Table](https://www.jianshu.com/p/1a94bb09a6e6)