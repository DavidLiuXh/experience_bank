*  Librdkafka将与kafka broker的交互,内部实现的一些操作,都封装成operator结构, 然后放入操作处理队列里, 统一处理;
*  这个队列其实也是一个线程间通讯的管道;
* 围绕这个队列的操作,是rdkafka的精华所在, 如果搞过windows编程的话,这个相当于event loop, 如果搞过golang的话,这个跟channel有点像;
*  包括如下结构介绍:
   1. rd_kafka_q_s
   2. d_kafka_queue_s
---
###### rd_kafka_q_s
* 所在文件: src/rdkafka_queue.h(c)
* 定义:
```
struct rd_kafka_q_s {
	mtx_t  rkq_lock; // 对队列操作的加锁用
	cnd_t  rkq_cond; // 队列中放入新的元素时, 用条件变量唤醒相应的等待线程

	struct rd_kafka_q_s *rkq_fwdq; /* Forwarded/Routed queue.
					* Used in place of this queue
					* for all operations. */

        // 放入队列的操作都存在这个tailq队列里
	struct rd_kafka_op_tailq rkq_q;  /* TAILQ_HEAD(, rd_kafka_op_s) */
	int           rkq_qlen;      /* Number of entries in queue */
        int64_t       rkq_qsize;     /* Size of all entries in queue */

        int           rkq_refcnt; //引用计数 
        int           rkq_flags; // 当前队列的状态,以下三个可选值
#define RD_KAFKA_Q_F_ALLOCATED  0x1  /* Allocated: rd_free on destroy */
#define RD_KAFKA_Q_F_READY      0x2  /* Queue is ready to be used.
                                      * Flag is cleared on destroy */
#define RD_KAFKA_Q_F_FWD_APP    0x4  /* Queue is being forwarded by a call
                                      * to rd_kafka_queue_forward. */

        rd_kafka_t   *rkq_rk; //此队列关联的rd_kafka_t句柄

        //队列中放入新的元素时, 用向fd写入数据的方式唤醒相应的等待线程
	struct rd_kafka_q_io *rkq_qio;   /* FD-based application signalling */

         // 队列中的操作被执行中所执行的回调函数
        /* Op serve callback (optional).
         * Mainly used for forwarded queues to use the original queue's
         * serve function from the forwarded position.
         * Shall return 1 if op was handled, else 0. */
        rd_kafka_q_serve_cb_t *rkq_serve;
        void *rkq_opaque;

       // 当前队列的名字,主要是调试用
#if ENABLE_DEVEL
	char rkq_name[64];       /* Debugging: queue name (FUNC:LINE) */
#else
	const char *rkq_name;    /* Debugging: queue name (FUNC) */
#endif
};
```
* 清除队列中所有的op元素 `rd_kafka_q_purge0`:
```
int rd_kafka_q_purge0 (rd_kafka_q_t *rkq, int do_lock) {
	rd_kafka_op_t *rko, *next;
	TAILQ_HEAD(, rd_kafka_op_s) tmpq = TAILQ_HEAD_INITIALIZER(tmpq);
        rd_kafka_q_t *fwdq;
        int cnt = 0;

        if (do_lock)
                mtx_lock(&rkq->rkq_lock);

        // 如果rkq_fwdq有值不为空, 就清除rkq_fwdq队列
        // rd_kafka_q_fwd_get这方法会将rkq_fwdq的引用计数加1
        if ((fwdq = rd_kafka_q_fwd_get(rkq, 0))) {
                if (do_lock)
                        mtx_unlock(&rkq->rkq_lock);
                cnt = rd_kafka_q_purge(fwdq);
                rd_kafka_q_destroy(fwdq);
                return cnt;
        }

	/* Move ops queue to tmpq to avoid lock-order issue
	 * by locks taken from rd_kafka_op_destroy(). */
       // 将rkq->rkq_q这个tailq的队列挂到tmpq这个临时tailq上面, 减少lock的时间
	TAILQ_MOVE(&tmpq, &rkq->rkq_q, rko_link);

	/* Zero out queue */
        rd_kafka_q_reset(rkq);

        if (do_lock)
                mtx_unlock(&rkq->rkq_lock);

	/* Destroy the ops */
        // destroy 队列中的每一个op
	next = TAILQ_FIRST(&tmpq);
	while ((rko = next)) {
		next = TAILQ_NEXT(next, rko_link);
		rd_kafka_op_destroy(rko);
                cnt++;
	}

        return cnt;
}
```
* 引用计数为0时销毁当前队列, `rd_kafka_q_destroy0`:
```
static RD_INLINE RD_UNUSED
void rd_kafka_q_destroy0 (rd_kafka_q_t *rkq, int disable) {
        int do_delete = 0;

        if (disable) {
                /* To avoid recursive locking (from ops being purged
                 * that reference this queue somehow),
                 * we disable the queue and purge it with individual
                 * locking. */
                rd_kafka_q_disable0(rkq, 1/*lock*/);
                // 清除队列中所有的op元素
                rd_kafka_q_purge0(rkq, 1/*lock*/);
        }

        mtx_lock(&rkq->rkq_lock);
        rd_kafka_assert(NULL, rkq->rkq_refcnt > 0);
        do_delete = !--rkq->rkq_refcnt;
        mtx_unlock(&rkq->rkq_lock);

        // 根据引用计数来确定是否调用rd_kafka_q_destroy_final
        if (unlikely(do_delete))
                rd_kafka_q_destroy_final(rkq);
}
```
* 设置或者消除forward queue:
```
void rd_kafka_q_fwd_set0 (rd_kafka_q_t *srcq, rd_kafka_q_t *destq,
                          int do_lock, int fwd_app) {

        if (do_lock)
                mtx_lock(&srcq->rkq_lock);
        if (fwd_app)
                srcq->rkq_flags |= RD_KAFKA_Q_F_FWD_APP;

        // 先destroy原有的rkq_fwdq
	if (srcq->rkq_fwdq) {
		rd_kafka_q_destroy(srcq->rkq_fwdq);
		srcq->rkq_fwdq = NULL;
	}
	if (destq) {
                //被forward的destq的引用计数加1
		rd_kafka_q_keep(destq);

		/* If rkq has ops in queue, append them to fwdq's queue.
		 * This is an irreversible operation. */
                if (srcq->rkq_qlen > 0) {
			rd_dassert(destq->rkq_flags & RD_KAFKA_Q_F_READY);
                        // 拼接destq和原有的srcq,  下面详细分析
			rd_kafka_q_concat(destq, srcq); 
		}

                //重新赋值rkq_fwdq
		srcq->rkq_fwdq = destq;
	}
        if (do_lock)
                mtx_unlock(&srcq->rkq_lock);
}
```
* 两队列拼接 `rd_kafka_q_concat0`:
```
int rd_kafka_q_concat0 (rd_kafka_q_t *rkq, rd_kafka_q_t *srcq, int do_lock) {
	int r = 0;

	while (srcq->rkq_fwdq) /* Resolve source queue */
		srcq = srcq->rkq_fwdq;
	if (unlikely(srcq->rkq_qlen == 0))
		return 0; /* Don't do anything if source queue is empty */

	if (do_lock)
		mtx_lock(&rkq->rkq_lock);
	if (!rkq->rkq_fwdq) {
                rd_kafka_op_t *rko;

                rd_dassert(TAILQ_EMPTY(&srcq->rkq_q) ||
                           srcq->rkq_qlen > 0);
		if (unlikely(!(rkq->rkq_flags & RD_KAFKA_Q_F_READY))) {
                        if (do_lock)
                                mtx_unlock(&rkq->rkq_lock);
			return -1;
		}
                /* First insert any prioritized ops from srcq
                 * in the right position in rkq. */
               // 先将srcq->rkq_q中rko的优先级>0的转移到rkq->rkq_q上, 并且按优先级插入适当位置
                while ((rko = TAILQ_FIRST(&srcq->rkq_q)) && rko->rko_prio > 0) {
                        TAILQ_REMOVE(&srcq->rkq_q, rko, rko_link);
                        TAILQ_INSERT_SORTED(&rkq->rkq_q, rko,
                                            rd_kafka_op_t *, rko_link,
                                            rd_kafka_op_cmp_prio);
                }

                // 将srcq->rkq_q中剩下的优先级=0的拼到rkq->rkq_q队尾
		TAILQ_CONCAT(&rkq->rkq_q, &srcq->rkq_q, rko_link);
		if (rkq->rkq_qlen == 0)
			rd_kafka_q_io_event(rkq);
                rkq->rkq_qlen += srcq->rkq_qlen;
                rkq->rkq_qsize += srcq->rkq_qsize;
		cnd_signal(&rkq->rkq_cond);

                rd_kafka_q_reset(srcq);
	} else
                //(rkq->rkq_fwdq ? rkq->rkq_fwdq : rkq这个有点多余, 直接rkq->rkq_fwdq 就成
		r = rd_kafka_q_concat0(rkq->rkq_fwdq ? rkq->rkq_fwdq : rkq,
				       srcq,
				       rkq->rkq_fwdq ? do_lock : 0);
	if (do_lock)
		mtx_unlock(&rkq->rkq_lock);

	return r;
}
```
* 内部函数,op元素插入队列`rd_kafka_q_enq0`:
```
rd_kafka_q_enq0 (rd_kafka_q_t *rkq, rd_kafka_op_t *rko, int at_head) {
    // 优先级=0, 放到队尾
    if (likely(!rko->rko_prio))
        TAILQ_INSERT_TAIL(&rkq->rkq_q, rko, rko_link);
    else if (at_head)
            // 强制插到头部
            TAILQ_INSERT_HEAD(&rkq->rkq_q, rko, rko_link);
    else
        // 按优先级确定位置 
        TAILQ_INSERT_SORTED(&rkq->rkq_q, rko, rd_kafka_op_t *,
                            rko_link, rd_kafka_op_cmp_prio);
    rkq->rkq_qlen++;
    rkq->rkq_qsize += rko->rko_len;
}
```
* op元素插入队列`rd_kafka_q_enq`:
```
int rd_kafka_q_enq (rd_kafka_q_t *rkq, rd_kafka_op_t *rko) {
	rd_kafka_q_t *fwdq;
	mtx_lock(&rkq->rkq_lock);
        rd_dassert(rkq->rkq_refcnt > 0);

        // 如果当前队列不是ready状态, 走rd_kafka_op_reply流程
        if (unlikely(!(rkq->rkq_flags & RD_KAFKA_Q_F_READY))) {
                /* Queue has been disabled, reply to and fail the rko. */
                mtx_unlock(&rkq->rkq_lock);

                return rd_kafka_op_reply(rko, RD_KAFKA_RESP_ERR__DESTROY);
        }

        // 如rko->rko_serve为null的话, 把默认的serve赋值给它
        if (!rko->rko_serve && rkq->rkq_serve) {
                /* Store original queue's serve callback and opaque
                 * prior to forwarding. */
                rko->rko_serve = rkq->rkq_serve;
                rko->rko_serve_opaque = rkq->rkq_opaque;
        }

        // 如果有rkq_fwdq队列,优先入rkq_fwdq队列, 没有直接入rkq_q队列
	if (!(fwdq = rd_kafka_q_fwd_get(rkq, 0))) {
		rd_kafka_q_enq0(rkq, rko, 0);
		cnd_signal(&rkq->rkq_cond);
		if (rkq->rkq_qlen == 1)
			rd_kafka_q_io_event(rkq);
		mtx_unlock(&rkq->rkq_lock);
	} else {
		mtx_unlock(&rkq->rkq_lock);
		rd_kafka_q_enq(fwdq, rko);
		rd_kafka_q_destroy(fwdq); //rd_kafka_q_fwd_get将引用计数+1,这里-1
	}

        return 1;
}
```
* `int rd_kafka_q_reenq (rd_kafka_q_t *rkq, rd_kafka_op_t *rko)`无锁版的`rd_kafka_q_enq`
* 清除rd_kafka_toppar_t->version过期的op元素`rd_kafka_q_purge_toppar_version`
```
void rd_kafka_q_purge_toppar_version (rd_kafka_q_t *rkq,
                                      rd_kafka_toppar_t *rktp, int version) {
	rd_kafka_op_t *rko, *next;
	TAILQ_HEAD(, rd_kafka_op_s) tmpq = TAILQ_HEAD_INITIALIZER(tmpq);
        int32_t cnt = 0;
        int64_t size = 0;
        rd_kafka_q_t *fwdq;

	mtx_lock(&rkq->rkq_lock);

       // 有rkq_fwdq就清除rkq_fwdq
        if ((fwdq = rd_kafka_q_fwd_get(rkq, 0))) {
                mtx_unlock(&rkq->rkq_lock);
                rd_kafka_q_purge_toppar_version(fwdq, rktp, version);
                rd_kafka_q_destroy(fwdq);
                return;
        }

        /* Move ops to temporary queue and then destroy them from there
         * without locks to avoid lock-ordering problems in op_destroy() */
       // 比较version, 收集需要清除的op
        while ((rko = TAILQ_FIRST(&rkq->rkq_q)) && rko->rko_rktp &&
               rd_kafka_toppar_s2i(rko->rko_rktp) == rktp &&
               rko->rko_version < version) {
                TAILQ_REMOVE(&rkq->rkq_q, rko, rko_link);
                TAILQ_INSERT_TAIL(&tmpq, rko, rko_link);
                cnt++;
                size += rko->rko_len;
        }


        rkq->rkq_qlen -= cnt;
        rkq->rkq_qsize -= size;
	mtx_unlock(&rkq->rkq_lock);

        /清除
	next = TAILQ_FIRST(&tmpq);
	while ((rko = next)) {
		next = TAILQ_NEXT(next, rko_link);
		rd_kafka_op_destroy(rko);
	}
}
```
* 最多处理队列中的一个op元素 `rd_kafka_q_pop_serve`, 先看有无符合条件(按version过滤)的op可处理, 没有则等待, 如果超时, 函数退出
```
rd_kafka_op_t *rd_kafka_q_pop_serve (rd_kafka_q_t *rkq, int timeout_ms,
                                     int32_t version,
                                     rd_kafka_q_cb_type_t cb_type,
                                     rd_kafka_q_serve_cb_t *callback,
                                     void *opaque) {
	rd_kafka_op_t *rko;
        rd_kafka_q_t *fwdq;

        rd_dassert(cb_type);

        // RD_POLL_INFINITE等同于一直等待,直到condition被signal
	if (timeout_ms == RD_POLL_INFINITE)
		timeout_ms = INT_MAX;

	mtx_lock(&rkq->rkq_lock);

        rd_kafka_yield_thread = 0;
        if (!(fwdq = rd_kafka_q_fwd_get(rkq, 0))) {
                do {
                        rd_kafka_op_res_t res;
                        rd_ts_t pre;

                        /* Filter out outdated ops */
                retry:
                        //找到符合条件的op, 按version来过滤
                        while ((rko = TAILQ_FIRST(&rkq->rkq_q)) &&
                               !(rko = rd_kafka_op_filter(rkq, rko, version)))
                                ;

                        if (rko) {
                                /* Proper versioned op */
                                // 将找到的op弹出队列
                                rd_kafka_q_deq0(rkq, rko);

                                /* Ops with callbacks are considered handled
                                 * and we move on to the next op, if any.
                                 * Ops w/o callbacks are returned immediately */
                                // 处理op
                                res = rd_kafka_op_handle(rkq->rkq_rk, rkq, rko,
                                                         cb_type, opaque,
                                                         callback);
                                if (res == RD_KAFKA_OP_RES_HANDLED)
                                        goto retry; /* Next op */
                                else if (unlikely(res ==
                                                  RD_KAFKA_OP_RES_YIELD)) {
                                        /* Callback yielded, unroll */
                                        mtx_unlock(&rkq->rkq_lock);
                                        return NULL;
                                } else
                                        break; /* Proper op, handle below. */
                        }

                        /* No op, wait for one if we have time left */
                        if (timeout_ms == RD_POLL_NOWAIT)
                                break;

			pre = rd_clock();
                        //执行wait操作
			if (cnd_timedwait_ms(&rkq->rkq_cond,
					     &rkq->rkq_lock,
					     timeout_ms) ==
			    thrd_timedout) {
				mtx_unlock(&rkq->rkq_lock);
				return NULL;
			}
			/* Remove spent time */
			timeout_ms -= (int) (rd_clock()-pre) / 1000;
			if (timeout_ms < 0)
				timeout_ms = RD_POLL_NOWAIT;

		} while (timeout_ms != RD_POLL_NOWAIT);

                mtx_unlock(&rkq->rkq_lock);

        } else {
                /* Since the q_pop may block we need to release the parent
                 * queue's lock. */
                mtx_unlock(&rkq->rkq_lock);
		rko = rd_kafka_q_pop_serve(fwdq, timeout_ms, version,
					   cb_type, callback, opaque);
                rd_kafka_q_destroy(fwdq);
        }
	return rko;
}
```
* 批量处理队列中的op, 无需按version过滤 `rd_kafka_q_serve`
```
int rd_kafka_q_serve (rd_kafka_q_t *rkq, int timeout_ms,
                      int max_cnt, rd_kafka_q_cb_type_t cb_type,
                      rd_kafka_q_serve_cb_t *callback, void *opaque) {
        rd_kafka_t *rk = rkq->rkq_rk;
	rd_kafka_op_t *rko;
	rd_kafka_q_t localq;
        rd_kafka_q_t *fwdq;
        int cnt = 0;

        rd_dassert(cb_type);

	mtx_lock(&rkq->rkq_lock);

        rd_dassert(TAILQ_EMPTY(&rkq->rkq_q) || rkq->rkq_qlen > 0);
        if ((fwdq = rd_kafka_q_fwd_get(rkq, 0))) {
                int ret;
                /* Since the q_pop may block we need to release the parent
                 * queue's lock. */
                mtx_unlock(&rkq->rkq_lock);
		ret = rd_kafka_q_serve(fwdq, timeout_ms, max_cnt,
                                       cb_type, callback, opaque);
                rd_kafka_q_destroy(fwdq);
		return ret;
	}

	if (timeout_ms == RD_POLL_INFINITE)
		timeout_ms = INT_MAX;

	/* Wait for op */
       // 如果当前队列为空, 就wait, 如果wait超时后队列还为空就直接退出
	while (!(rko = TAILQ_FIRST(&rkq->rkq_q)) && timeout_ms != 0) {
		if (cnd_timedwait_ms(&rkq->rkq_cond,
				     &rkq->rkq_lock,
				     timeout_ms) != thrd_success)
			break;

		timeout_ms = 0;
	}

	if (!rko) {
		mtx_unlock(&rkq->rkq_lock);
		return 0;
	}

       //将需要处理的op转移到临时的localq中
	/* Move the first `max_cnt` ops. */
	rd_kafka_q_init(&localq, rkq->rkq_rk);
	rd_kafka_q_move_cnt(&localq, rkq, max_cnt == 0 ? -1/*all*/ : max_cnt,
			    0/*no-locks*/);

        mtx_unlock(&rkq->rkq_lock);

        rd_kafka_yield_thread = 0;

	/* Call callback for each op */
       // 处理op
        while ((rko = TAILQ_FIRST(&localq.rkq_q))) {
                rd_kafka_op_res_t res;

                rd_kafka_q_deq0(&localq, rko);
                res = rd_kafka_op_handle(rk, &localq, rko, cb_type,
                                         opaque, callback);
                /* op must have been handled */
                rd_kafka_assert(NULL, res != RD_KAFKA_OP_RES_PASS);
                cnt++;

                if (unlikely(res == RD_KAFKA_OP_RES_YIELD ||
                             rd_kafka_yield_thread)) {
                        /* Callback called rd_kafka_yield(), we must
                         * stop our callback dispatching and put the
                         * ops in localq back on the original queue head. */
                        // 当前thread资源被出让, 没处理完的op要还回到rkq中
                        if (!TAILQ_EMPTY(&localq.rkq_q))
                                rd_kafka_q_prepend(rkq, &localq);
                        break;
                }
	}

	rd_kafka_q_destroy_owner(&localq);

	return cnt;
}
```
* 等待处理单个op:
```
rd_kafka_resp_err_t rd_kafka_q_wait_result (rd_kafka_q_t *rkq, int timeout_ms) {
        rd_kafka_op_t *rko;
        rd_kafka_resp_err_t err;

        rko = rd_kafka_q_pop(rkq, timeout_ms, 0);
        if (!rko)
                err = RD_KAFKA_RESP_ERR__TIMED_OUT;
        else {
                err = rko->rko_err;
                rd_kafka_op_destroy(rko);
        }

        return err;
}
```
* 处理 fetch操作, 即获取kafka message `rd_kafka_q_serve_rkmessages`:
```
int rd_kafka_q_serve_rkmessages (rd_kafka_q_t *rkq, int timeout_ms,
                                 rd_kafka_message_t **rkmessages,
                                 size_t rkmessages_size) {
	unsigned int cnt = 0;
        TAILQ_HEAD(, rd_kafka_op_s) tmpq = TAILQ_HEAD_INITIALIZER(tmpq);
        rd_kafka_op_t *rko, *next;
        rd_kafka_t *rk = rkq->rkq_rk;
        rd_kafka_q_t *fwdq;

	mtx_lock(&rkq->rkq_lock);
        // 如果有 forward queue就处理之
        if ((fwdq = rd_kafka_q_fwd_get(rkq, 0))) {
                /* Since the q_pop may block we need to release the parent
                 * queue's lock. */
                mtx_unlock(&rkq->rkq_lock);
		cnt = rd_kafka_q_serve_rkmessages(fwdq, timeout_ms,
						  rkmessages, rkmessages_size);
                rd_kafka_q_destroy(fwdq);
		return cnt;
	}
        mtx_unlock(&rkq->rkq_lock);

        rd_kafka_yield_thread = 0;
	while (cnt < rkmessages_size) {
                rd_kafka_op_res_t res;

                mtx_lock(&rkq->rkq_lock);

               // 如果rkq_q为空, 就wait, 如果超时,退出
		while (!(rko = TAILQ_FIRST(&rkq->rkq_q))) {
			if (cnd_timedwait_ms(&rkq->rkq_cond, &rkq->rkq_lock,
                                             timeout_ms) == thrd_timedout)
				break;
		}

		if (!rko) {
                        mtx_unlock(&rkq->rkq_lock);
			break; /* Timed out */
                }

                // op弹出队列
		rd_kafka_q_deq0(rkq, rko);

                mtx_unlock(&rkq->rkq_lock);

                // 判断op是否过期,过期的话加入到tmpq, 之后统一清除
		if (rd_kafka_op_version_outdated(rko, 0)) {
                        /* Outdated op, put on discard queue */
                        TAILQ_INSERT_TAIL(&tmpq, rko, rko_link);
                        continue;
                }

                /* Serve non-FETCH callbacks */
                res = rd_kafka_poll_cb(rk, rkq, rko,
                                       RD_KAFKA_Q_CB_RETURN, NULL);
                if (res == RD_KAFKA_OP_RES_HANDLED) {
                        /* Callback served, rko is destroyed. */
                        continue;
                } else if (unlikely(res == RD_KAFKA_OP_RES_YIELD ||
                                    rd_kafka_yield_thread)) {
                        /* Yield. */
                        break;
                }
                rd_dassert(res == RD_KAFKA_OP_RES_PASS);

		/* Auto-commit offset, if enabled. */
		if (!rko->rko_err && rko->rko_type == RD_KAFKA_OP_FETCH) {
                        rd_kafka_toppar_t *rktp;
                        rktp = rd_kafka_toppar_s2i(rko->rko_rktp);
			rd_kafka_toppar_lock(rktp);
			rktp->rktp_app_offset = rko->rko_u.fetch.rkm.rkm_offset+1;
                        if (rktp->rktp_cgrp &&
			    rk->rk_conf.enable_auto_offset_store)
                                //存储offset, auto-commit时用
                                rd_kafka_offset_store0(rktp,
						       rktp->rktp_app_offset,
                                                       0/* no lock */);
			rd_kafka_toppar_unlock(rktp);
                }

		/* Get rkmessage from rko and append to array. */
		rkmessages[cnt++] = rd_kafka_message_get(rko);
	}

        /* Discard non-desired and already handled ops */
        next = TAILQ_FIRST(&tmpq);
        while (next) {
                rko = next;
                next = TAILQ_NEXT(next, rko_link);
                rd_kafka_op_destroy(rko);
        }

	return cnt;
}
```
###### rd_kafka_queue_s
* 所在文件: src/rdkafka_queue.h
* 这个是对外提供服务的接口,以rd_kafka_t和rd_kafka_q_t组合在一起
* 定义:
```
struct rd_kafka_queue_s {
	rd_kafka_q_t *rkqu_q;
        rd_kafka_t   *rkqu_rk;
};
```
* 创建 `rd_kafka_queue_s`:
```
rd_kafka_queue_t *rd_kafka_queue_new (rd_kafka_t *rk) {
	rd_kafka_q_t *rkq;
	rd_kafka_queue_t *rkqu;

	rkq = rd_kafka_q_new(rk);
	rkqu = rd_kafka_queue_new0(rk, rkq);
	rd_kafka_q_destroy(rkq); /* Loose refcount from q_new, one is held
				  * by queue_new0 */
	return rkqu;
}
```
* 获取rdkafka和应用层交互使用的 `rd_kafka_queue_s`:
```
rd_kafka_queue_t *rd_kafka_queue_get_main (rd_kafka_t *rk) {
	return rd_kafka_queue_new0(rk, rk->rk_rep);
}
```
* 获取consumer对应的 `rd_kafka_queue_s`:
```
rd_kafka_queue_t *rd_kafka_queue_get_consumer (rd_kafka_t *rk) {
	if (!rk->rk_cgrp)
		return NULL;
	return rd_kafka_queue_new0(rk, rk->rk_cgrp->rkcg_q);
}
```
* 获取任一partition对应的`rd_kafka_queue_s`:
```
rd_kafka_queue_t *rd_kafka_queue_get_partition (rd_kafka_t *rk,
                                                const char *topic,
                                                int32_t partition) {
        shptr_rd_kafka_toppar_t *s_rktp;
        rd_kafka_toppar_t *rktp;
        rd_kafka_queue_t *result;

        if (rk->rk_type == RD_KAFKA_PRODUCER)
                return NULL;

        s_rktp = rd_kafka_toppar_get2(rk, topic,
                                      partition,
                                      0, /* no ua_on_miss */
                                      1 /* create_on_miss */);

        if (!s_rktp)
                return NULL;

        rktp = rd_kafka_toppar_s2i(s_rktp);
        result = rd_kafka_queue_new0(rk, rktp->rktp_fetchq);
        rd_kafka_toppar_destroy(s_rktp);

        return result;
}
```
---
### [Librdkafka源码分析-Content Table](https://www.jianshu.com/p/1a94bb09a6e6)