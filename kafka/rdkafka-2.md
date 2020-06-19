* Timer
* 原子操作
* 引用计数
---
###### Timer
* 所在文件: sr/rdkafka_timer.c(h)
* 主要是通过TimerManager来管理多个timer, 达到处理定时任务的效果
* TimerManager定义:
```
typedef struct rd_kafka_timers_s {
        TAILQ_HEAD(, rd_kafka_timer_s) rkts_timers;
        struct rd_kafka_s *rkts_rk;
	mtx_t       rkts_lock;
	cnd_t       rkts_cond;

        int         rkts_enabled;
} rd_kafka_timers_t;
```
  1. 使用`TAILQ`来管理多个timer, 这个 队列是个有序队列, 按` rd_kafka_timer_s`中的`rtmr_next`从小到大排列;
  2. 对timer 队列的操作需要加锁保护: `rkts_lock`
* Timer定义:
```
typedef struct rd_kafka_timer_s {
	TAILQ_ENTRY(rd_kafka_timer_s)  rtmr_link;

	rd_ts_t rtmr_next;
	rd_ts_t rtmr_interval;   /* interval in microseconds */

	void  (*rtmr_callback) (rd_kafka_timers_t *rkts, void *arg);
	void   *rtmr_arg;
} rd_kafka_timer_t;
```
 1. `rtmr_link` : `TAILQ`元素
 2. `rtmr_next`: 当前timer的下一次到期时间, 绝对时间;
 3. `rtmr_interval`: 执行的时间间隔;
 4. `rtmr_callback`: 时期时执行的回调函数;
* 加入新的timer到TimerManager中:
```
void rd_kafka_timer_start (rd_kafka_timers_t *rkts,
			   rd_kafka_timer_t *rtmr, rd_ts_t interval,
			   void (*callback) (rd_kafka_timers_t *rkts, void *arg),
			   void *arg) {
	rd_kafka_timers_lock(rkts);
	rd_kafka_timer_stop(rkts, rtmr, 0/*!lock*/); 

	rtmr->rtmr_interval = interval;
	rtmr->rtmr_callback = callback;
	rtmr->rtmr_arg      = arg;

	rd_kafka_timer_schedule(rkts, rtmr, 0);
	rd_kafka_timers_unlock(rkts);
}
```
  1. 此timer已经在队列中的话,要先stop;
  2. 重新设置 timer的各参数;
  3. 加入队列;
*  Timer的插入: 根据`rtmr_next`值在队列中找到合适的位置后插入;
```
static void rd_kafka_timer_schedule (rd_kafka_timers_t *rkts,
				     rd_kafka_timer_t *rtmr, int extra_us) {
	rd_kafka_timer_t *first;

	/* Timer has been stopped */
	if (!rtmr->rtmr_interval)
		return;

        /* Timers framework is terminating */
        if (unlikely(!rkts->rkts_enabled))
                return;

	rtmr->rtmr_next = rd_clock() + rtmr->rtmr_interval + extra_us;

	if (!(first = TAILQ_FIRST(&rkts->rkts_timers)) ||
	    first->rtmr_next > rtmr->rtmr_next) {
		TAILQ_INSERT_HEAD(&rkts->rkts_timers, rtmr, rtmr_link);
                cnd_signal(&rkts->rkts_cond);
	} else
		TAILQ_INSERT_SORTED(&rkts->rkts_timers, rtmr,
                                    rd_kafka_timer_t *, rtmr_link,
				    rd_kafka_timer_cmp);
}
```
* Timer的调度执行:
```
void rd_kafka_timers_run (rd_kafka_timers_t *rkts, int timeout_us) {
	rd_ts_t now = rd_clock();
	rd_ts_t end = now + timeout_us;

        rd_kafka_timers_lock(rkts);

	while (!rd_atomic32_get(&rkts->rkts_rk->rk_terminate) && now <= end) {
		int64_t sleeptime;
		rd_kafka_timer_t *rtmr;

		if (timeout_us != RD_POLL_NOWAIT) {
			sleeptime = rd_kafka_timers_next(rkts,
							 timeout_us,
							 0/*no-lock*/);

			if (sleeptime > 0) {
				cnd_timedwait_ms(&rkts->rkts_cond,
						 &rkts->rkts_lock,
						 (int)(sleeptime / 1000));

			}
		}

		now = rd_clock();

		while ((rtmr = TAILQ_FIRST(&rkts->rkts_timers)) &&
		       rtmr->rtmr_next <= now) {

			rd_kafka_timer_unschedule(rkts, rtmr);
                        rd_kafka_timers_unlock(rkts);

			rtmr->rtmr_callback(rkts, rtmr->rtmr_arg);

                        rd_kafka_timers_lock(rkts);
			/* Restart timer, unless it has been stopped, or
			 * already reschedueld (start()ed) from callback. */
			if (rd_kafka_timer_started(rtmr) &&
			    !rd_kafka_timer_scheduled(rtmr))
				rd_kafka_timer_schedule(rkts, rtmr, 0);
		}

		if (timeout_us == RD_POLL_NOWAIT) {
			/* Only iterate once, even if rd_clock doesn't change */
			break;
		}
	}

	rd_kafka_timers_unlock(rkts);
}
```
  1.  通过 `rd_kafka_timers_next`获取需要wait的时间 ;
  2. 需要wait就 `cnd_timedwait_ms`;
  3. 执行到期 timer的回调函数, 根据需要将此timer再次加入队列;
###### 原子操作
* 所在文件: src/rdatomic.h
* 如果当前GCC支持__atomic_组操作,就使用GCC的[build-in函数](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html)
* 如果不支持, 原子操作用锁来模拟实现;
* 在Windows上用`Interlocked`族函数实现;
###### 引用计数
*  所在文件: src/rd.h
* 定义:
```
#ifdef RD_REFCNT_USE_LOCKS
typedef struct rd_refcnt_t {
        mtx_t lock;
        int v;
} rd_refcnt_t;
#else
typedef rd_atomic32_t rd_refcnt_t;
#endif
```
 1. 由定义我们可以看出可以通过锁来实现,也可以通过上面介绍的原子类型来实现这个计数;
* 引用计数的操作接口, 也是分成了锁(实现成函数)和原子类型(实现成宏)两种不同的实现
```
static RD_INLINE RD_UNUSED int rd_refcnt_init (rd_refcnt_t *R, int v)
static RD_INLINE RD_UNUSED void rd_refcnt_destroy (rd_refcnt_t *R)
static RD_INLINE RD_UNUSED int rd_refcnt_set (rd_refcnt_t *R, int v) 
static RD_INLINE RD_UNUSED int rd_refcnt_add0 (rd_refcnt_t *R)
static RD_INLINE RD_UNUSED int rd_refcnt_sub0 (rd_refcnt_t *R)
static RD_INLINE RD_UNUSED int rd_refcnt_get (rd_refcnt_t *R)
```
###### 智能指针
* 所在文件: src/rd.h
* 智能指针就是加了上面的引用计数的指针
* 定义:
```
#define RD_SHARED_PTR_TYPE(STRUCT_NAME,WRAPPED_TYPE) WRAPPED_TYPE
//get的同时会将此用计数 +1
#define rd_shared_ptr_get_src(FUNC,LINE,OBJ,REFCNT,SPTR_TYPE)	\
        (rd_refcnt_add(REFCNT), (OBJ))
#define rd_shared_ptr_get(OBJ,REFCNT,SPTR_TYPE)          \
        (rd_refcnt_add(REFCNT), (OBJ))

#define rd_shared_ptr_obj(SPTR) (SPTR)

// put使用rd_refcnt_destroywrapper实现, 引用计数减为0,则调用DESTRUCTOR作清理释放
#define rd_shared_ptr_put(SPTR,REF,DESTRUCTOR)                  \
                rd_refcnt_destroywrapper(REF,DESTRUCTOR)
```
* 在C中实现引用计数, 哪里要+1, 哪里要-1, 全凭使用者自己根据代码逻辑需要来控制,因此很容易导致少+1, 多+1, 少-1, 多-1的情况, 因此rdkafka作者又提供了一个debug版本的实现, 跟踪了调用函数, 所在行等信息, 供调试排查问题用,其实实现也很简单, 但还是比较巧妙的
```
#define RD_SHARED_PTR_TYPE(STRUCT_NAME, WRAPPED_TYPE) \
        struct STRUCT_NAME {                          \
                LIST_ENTRY(rd_shptr0_s) link;         \
                WRAPPED_TYPE *obj;                     \
                rd_refcnt_t *ref;                     \
                const char *typename;                 \
                const char *func;                     \
                int line;                             \
        }
/* Common backing struct compatible with RD_SHARED_PTR_TYPE() types */
typedef RD_SHARED_PTR_TYPE(rd_shptr0_s, void) rd_shptr0_t;

LIST_HEAD(rd_shptr0_head, rd_shptr0_s);
extern struct rd_shptr0_head rd_shared_ptr_debug_list;
extern mtx_t rd_shared_ptr_debug_mtx;
```
引用了一个新的struct来将引用计数和调用信息结合起来, 使用链表来管理这个struct的对象. 每次对引用计数的操作都要操作这个链表.
```
static RD_INLINE RD_UNUSED RD_WARN_UNUSED_RESULT __attribute__((warn_unused_result))
rd_shptr0_t *rd_shared_ptr_get0 (const char *func, int line,
                                 const char *typename,
                                 rd_refcnt_t *ref, void *obj) {
        //创建shared ptr struct结构
        rd_shptr0_t *sptr = rd_calloc(1, sizeof(*sptr));
        sptr->obj = obj;
        sptr->ref = ref;
        sptr->typename = typename;
        sptr->func = func;
        sptr->line = line;

       //加入链表
        mtx_lock(&rd_shared_ptr_debug_mtx);
        LIST_INSERT_HEAD(&rd_shared_ptr_debug_list, sptr, link);
        mtx_unlock(&rd_shared_ptr_debug_mtx);
        return sptr;
}

#define rd_shared_ptr_put(SPTR,REF,DESTRUCTOR) do {               \
               // 引用计数 -1, 到0话清理释放
                if (rd_refcnt_sub(REF) == 0)                      \
                        DESTRUCTOR;                               \
                mtx_lock(&rd_shared_ptr_debug_mtx);               \
                //从链表中移除struct 对象
                LIST_REMOVE(SPTR, link);                          \
                mtx_unlock(&rd_shared_ptr_debug_mtx);             \
                rd_free(SPTR);                                    \
        } while (0)
```
---
### [Librdkafka源码分析-Content Table](https://www.jianshu.com/p/1a94bb09a6e6)