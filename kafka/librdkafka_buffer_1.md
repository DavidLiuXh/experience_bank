* Temporary buffer
* Buffer segment
* Buffer
* Buffer slice
---
###### 临时写入缓冲区 
* 所在文件: src/rdkafka_buf.h(c)
* 写操作缓冲区, 写入时保证了8位内存对齐, 以便提高内存读取效率和跨平台的安全性;
* 定义:
```
typedef struct rd_tmpabuf_s {
	size_t size; //buf即内部缓冲区容量
	size_t of; //当前写入位置
	char  *buf; //内部缓冲区
	int    failed; //写入是否发生了错误
	int    assert_on_fail; //写入发生错误时,是否assert
} rd_tmpabuf_t;
```
* 初始化:
```
static RD_UNUSED void
rd_tmpabuf_new (rd_tmpabuf_t *tab, size_t size, int assert_on_fail) {
	tab->buf = rd_malloc(size); //分配内部缓冲区
	tab->size = size; //buf即内部缓冲区容量
	tab->of = 0;
	tab->failed = 0;
	tab->assert_on_fail = assert_on_fail;
}
```
* 根据写入大小返回可供写入的内存位置:
```
static RD_UNUSED void *
rd_tmpabuf_alloc0 (const char *func, int line, rd_tmpabuf_t *tab, size_t size) {
	void *ptr;

	if (unlikely(tab->failed)) //发错标志已设置, 则直接返回NULL
		return NULL;

	if (unlikely(tab->of + size > tab->size)) { //请求的内存size超出缓冲区容量, 作错误处理
		if (tab->assert_on_fail) {
			fprintf(stderr,
				"%s: %s:%d: requested size %zd + %zd > %zd\n",
				__FUNCTION__, func, line, tab->of, size,
				tab->size);
			assert(!*"rd_tmpabuf_alloc: not enough size in buffer");
		}
		return NULL;
	}

        ptr = (void *)(tab->buf + tab->of); //获取可供写入的内存位转置:
	tab->of += RD_ROUNDUP(size, 8); //更新当前写入位置, 并作8位对齐

	return ptr;
}
```
* buffer写入:
```
static RD_UNUSED void *
rd_tmpabuf_write0 (const char *func, int line,
		   rd_tmpabuf_t *tab, const void *buf, size_t size) {
	void *ptr = rd_tmpabuf_alloc0(func, line, tab, size); //根据写入大小返回可供写入的内存位置

	if (ptr)
		memcpy(ptr, buf, size); //内存copy

	return ptr;
}
```
###### Buffer segment
* 所在文件: src/rdbuf.c(h)
* 顾名思义, `buffer segmeng`和`buffer`的组成部分, `buffer`是我们下一节要介绍的`rd_buf_t`;
* 定义:
```
typedef struct rd_segment_s {
        TAILQ_ENTRY(rd_segment_s)  seg_link; /*<< rbuf_segments Link */ tailq元素
        char  *seg_p;                /**< Backing-store memory */ 用于存储数据的内存指针
        size_t seg_of;               /**< Current relative write-position 在当前的segment中的写入位置
                                      *   (length of payload in this segment) */
        size_t seg_size;             /**< Allocated size of seg_p */ 存储数据的内存大小
        size_t seg_absof;            /**< Absolute offset of this segment's
                                      *   beginning in the grand rd_buf_t */ 当前segment 在整个buffer中写入的起始地址
        void (*seg_free) (void *p);  /**< Optional free function for seg_p */
        int    seg_flags;            /**< Segment flags */
#define RD_SEGMENT_F_RDONLY   0x1    /**< Read-only segment */
#define RD_SEGMENT_F_FREE     0x2    /**< Free segment on destroy,
                                      *   e.g, not a fixed segment. */
} rd_segment_t;
```
* 初始化:
```
static void rd_segment_init (rd_segment_t *seg, void *mem, size_t size) {
        memset(seg, 0, sizeof(*seg));
        seg->seg_p     = mem;
        seg->seg_size  = size;
}
```
* 销毁:
```
static void rd_segment_destroy (rd_segment_t *seg) {
        /* Free payload */
        //如果提供了seg_free, 也destroy seg-
        if (seg->seg_free && seg->seg_p)
                seg->seg_free(seg->seg_p);

        if (seg->seg_flags & RD_SEGMENT_F_FREE)
                rd_free(seg);
}
```
* 判断当前segment是否还有剩余空间, 并返回当前写指针, 返回值表示当前还可写入的大小
```
static RD_INLINE RD_UNUSED size_t
rd_segment_write_remains (const rd_segment_t *seg, void **p) {
        if (unlikely((seg->seg_flags & RD_SEGMENT_F_RDONLY)))
                return 0;
        if (p)
                *p = (void *)(seg->seg_p + seg->seg_of);
        return seg->seg_size - seg->seg_of;
}
```
###### 主角 Buffter
* 所在文件: src/rdbuf.c(h)
* 数据Buffer, 内包一个`rd_segment_t`的list
* 定义:
```
typedef struct rd_buf_s {
        struct rd_segment_head rbuf_segments; /**< TAILQ list of segments */ segment tailq的头指针
        size_t            rbuf_segment_cnt;   /**< Number of segments */ segment个数

        rd_segment_t     *rbuf_wpos;          /**< Current write position seg */ 当前正在写入的segment
        size_t            rbuf_len;           /**< Current (written) length */ 当前写入数据的总大小
        size_t            rbuf_size;          /**< Total allocated size of 
                                               *   all segments. */所有segment的存储空间大小

        char             *rbuf_extra;         /* Extra memory allocated for
                                               * use by segment structs,
                                               * buffer memory, etc. */ 可以理解成一个预分配的内存池
        size_t            rbuf_extra_len;     /* Current extra memory used */
        size_t            rbuf_extra_size;    /* Total size of extra memory */
} rd_buf_t;
```
* 确保当前写入的segment可以写入size大小的数据:
```
void rd_buf_write_ensure_contig (rd_buf_t *rbuf, size_t size) {
        rd_segment_t *seg = rbuf->rbuf_wpos;

        if (seg) {
                void *p;
                size_t remains = rd_segment_write_remains(seg, &p);

                // 如果当前正在写入的segment足够的剩余空间,那么就写入当前这个segment
                if (remains >= size)
                        return; /* Existing segment has enough space. */

                /* Future optimization:
                 * If existing segment has enough remaining space to warrant
                 * a split, do it, before allocating a new one. */
        }

        /* Allocate new segment */
        // 分配新的segment作为新的当前写入的segment
        rbuf->rbuf_wpos = rd_buf_alloc_segment(rbuf, size, size);
}
```
* 根据指定的offset来获取其所在的segment:
```
rd_segment_t *
rd_buf_get_segment_at_offset (const rd_buf_t *rbuf, const rd_segment_t *hint,
                              size_t absof) {
        // hint是为了少遍历segment list, 给定一个offset可以命中的segment
        const rd_segment_t *seg = hint;

        if (unlikely(absof > rbuf->rbuf_len))
                return NULL;

        /* Only use current write position if possible and if it helps */
        if (!seg || absof < seg->seg_absof)
                seg = TAILQ_FIRST(&rbuf->rbuf_segments);

        // 遍历查找
        do {
                if (absof >= seg->seg_absof &&
                    absof < seg->seg_absof + seg->seg_of) {
                        rd_dassert(seg->seg_absof <= rd_buf_len(rbuf));
                        return (rd_segment_t *)seg;
                }
        } while ((seg = TAILQ_NEXT(seg, seg_link)));

        return NULL;
}
```
* 将segment按指定的offset分离成一个单独的segment, 这是一个内部函数,按源码里的注释目前只用于最后一个segment的空闲存储空间分离
```
static rd_segment_t *rd_segment_split (rd_buf_t *rbuf, rd_segment_t *seg,
                                       size_t absof) {
        rd_segment_t *newseg;
        size_t relof;

        rd_assert(seg == rbuf->rbuf_wpos);
        rd_assert(absof >= seg->seg_absof &&
                  absof <= seg->seg_absof + seg->seg_of);

       // 确定在segment中的分离位置
        relof = absof - seg->seg_absof;

      //  创建新的segment
        newseg = rd_buf_alloc_segment0(rbuf, 0);

        /* Add later part of split bytes to new segment */
        newseg->seg_p      = seg->seg_p+relof;
        newseg->seg_of     = seg->seg_of-relof;
        newseg->seg_size   = seg->seg_size-relof;
        newseg->seg_absof  = SIZE_MAX; /* Invalid */
        newseg->seg_flags |= seg->seg_flags;

        /* Remove earlier part of split bytes from previous segment */
        // 原有segment更新
        seg->seg_of        = relof;
        seg->seg_size      = relof;

        /* newseg's length will be added to rbuf_len in append_segment(),
         * so shave it off here from seg's perspective. */
        // 分离后不再属于当前buffer, buffer相关值更新
        rbuf->rbuf_len   -= newseg->seg_of;
        rbuf->rbuf_size  -= newseg->seg_size;

        return newseg;
}
``` 
* buffer 初始化:
```
void rd_buf_init (rd_buf_t *rbuf, size_t fixed_seg_cnt, size_t buf_size) {
        size_t totalloc = 0;

        // 各成员清0
        memset(rbuf, 0, sizeof(*rbuf));

        // segment list初始化
        TAILQ_INIT(&rbuf->rbuf_segments);

        if (!fixed_seg_cnt) {
                assert(!buf_size);
                return;
        }

        /* Pre-allocate memory for a fixed set of segments that are known
         * before-hand, to minimize the number of extra allocations
         * needed for well-known layouts (such as headers, etc) */
        totalloc += RD_ROUNDUP(sizeof(rd_segment_t), 8) * fixed_seg_cnt;

        /* Pre-allocate extra space for the backing buffer. */
        totalloc += buf_size;

        rbuf->rbuf_extra_size = totalloc;

        // 预分配内存
        rbuf->rbuf_extra = rd_malloc(rbuf->rbuf_extra_size);
}
```
* 获取当前可用于写入数据的内存指针, 返回值是获取的写入指针所在segment的剩余空间
```
static size_t
rd_buf_get_writable0 (rd_buf_t *rbuf, rd_segment_t **segp, void **p) {
        rd_segment_t *seg;

        for (seg = rbuf->rbuf_wpos ; seg ; seg = TAILQ_NEXT(seg, seg_link)) {
                size_t len = rd_segment_write_remains(seg, p);

                /* Even though the write offset hasn't changed we
                 * avoid future segment scans by adjusting the
                 * wpos here to the first writable segment. */
               // 更新buffer的rbuf_wpos
                rbuf->rbuf_wpos = seg;
                if (segp)
                        *segp = seg;

                if (unlikely(len == 0))
                        continue;

                /* Also adjust absof if the segment was allocated
                 * before the previous segment's memory was exhausted
                 * and thus now might have a lower absolute offset
                 * than the previos segment's now higher relative offset. */
                // 更新获取的segment的seg_absof
                if (seg->seg_of == 0 && seg->seg_absof < rbuf->rbuf_len)
                        seg->seg_absof = rbuf->rbuf_len;

                return len;
        }

        return 0;
}
```
* 写payload到buffer:
```
size_t rd_buf_write (rd_buf_t *rbuf, const void *payload, size_t size) {
        size_t remains = size;
        size_t initial_absof;
        const char *psrc = (const char *)payload;

        initial_absof = rbuf->rbuf_len;

        /* Ensure enough space by pre-allocating segments. */
        rd_buf_write_ensure(rbuf, size, 0);

       // 循环写,直到payload全部写入
        while (remains > 0) {
                void *p;
                rd_segment_t *seg;
                // 获取当前可以用于写入的segment, 这个segment对应的内存指针和剩余空间
                size_t segremains = rd_buf_get_writable0(rbuf, &seg, &p);
                size_t wlen = RD_MIN(remains, segremains);

                rd_dassert(seg == rbuf->rbuf_wpos);
                rd_dassert(wlen > 0);
                rd_dassert(seg->seg_p+seg->seg_of <= (char *)p &&
                           (char *)p < seg->seg_p+seg->seg_size);

                if (payload) {
                        // payload内存copy
                        memcpy(p, psrc, wlen);
                        psrc += wlen;
                }

                // 更新各offset
                seg->seg_of    += wlen;
                rbuf->rbuf_len += wlen;
                remains        -= wlen;
        }

        rd_assert(remains == 0);

        return initial_absof;
}
```
* 从指定offset开始局部更新buffer
```
size_t rd_buf_write_update (rd_buf_t *rbuf, size_t absof,
                            const void *payload, size_t size) {
        rd_segment_t *seg;
        const char *psrc = (const char *)payload;
        size_t of;

        /* Find segment for offset */
       // 根据指定的offsewt找到需要更新的segment
        seg = rd_buf_get_segment_at_offset(rbuf, rbuf->rbuf_wpos, absof);
        rd_assert(seg && *"invalid absolute offset");

        // 可能需要更新多个连续的segment
        for (of = 0 ; of < size ; seg = TAILQ_NEXT(seg, seg_link)) {
                rd_assert(seg->seg_absof <= rd_buf_len(rbuf));
                // 更新
                size_t wlen = rd_segment_write_update(seg, absof+of,
                                                      psrc+of, size-of);
                of += wlen;
        }

        rd_dassert(of == size);

        return of;
}
```
* buffer的push操作
```
void rd_buf_push (rd_buf_t *rbuf, const void *payload, size_t size,
                  void (*free_cb)(void *)) {
        rd_segment_t *prevseg, *seg, *tailseg = NULL;

       // push操作前需要将当前正在写入的segment的剩余空间分离为一个独立的segment
        if ((prevseg = rbuf->rbuf_wpos) &&
            rd_segment_write_remains(prevseg, NULL) > 0) {
                /* If the current segment still has room in it split it
                 * and insert the pushed segment in the middle (below). */
                tailseg = rd_segment_split(rbuf, prevseg,
                                           prevseg->seg_absof +
                                           prevseg->seg_of);
        }

        // 创建一个新的segment, 用payload初始化,然后apeend到segment list
        seg = rd_buf_alloc_segment0(rbuf, 0);
        seg->seg_p      = (char *)payload;
        seg->seg_size   = size;
        seg->seg_of     = size;
        seg->seg_free   = free_cb;
        seg->seg_flags |= RD_SEGMENT_F_RDONLY;

        rd_buf_append_segment(rbuf, seg);

        // 将上面分离出来的独立的segment添加到segment list的末尾
        if (tailseg)
                rd_buf_append_segment(rbuf, tailseg);
}
```
* buffer的seek操作, 其实就是一个truncat操作
```
int rd_buf_write_seek (rd_buf_t *rbuf, size_t absof) {
        rd_segment_t *seg, *next;
        size_t relof;

       // 定位到需要操作的起始segment
        seg = rd_buf_get_segment_at_offset(rbuf, rbuf->rbuf_wpos, absof);
        if (unlikely(!seg))
                return -1;

        relof = absof - seg->seg_absof;
        if (unlikely(relof > seg->seg_of))
                return -1;

        /* Destroy sub-sequent segments in reverse order so that
         * destroy_segment() length checks are correct.
         * Will decrement rbuf_len et.al. */
        // 截断操作,destroy掉不再需要的segment
        for (next = TAILQ_LAST(&rbuf->rbuf_segments, rd_segment_head) ;
             next != seg ; next = TAILQ_PREV(next, rd_segment_head, seg_link))
                rd_buf_destroy_segment(rbuf, next);

        /* Update relative write offset */
        seg->seg_of         = relof;
        rbuf->rbuf_wpos     = seg;
        rbuf->rbuf_len      = seg->seg_absof + seg->seg_of;

        rd_assert(rbuf->rbuf_len == absof);

        return 0;
}
```
###### Buffer slice
* 所在文件: src/rdkafka_buf.h(c)
* 顾名思义, 表示上面介绍的 `rd_buffer_t`的一个片断, 只读, 不会修改buffer的内容;
* 定义:
```
typedef struct rd_slice_s {
        const rd_buf_t     *buf;    /**< Pointer to buffer */ 映射的rd_buffer_t
        const rd_segment_t *seg;    /**< Current read position segment.
                                     *   Will point to NULL when end of
                                     *   slice is reached. */ 当前正在读取的rd_buffer_t中的segment
        size_t              rof;    /**< Relative read offset in segment */  当前正在读取的rd_buffer_t中的segment的读offset
        size_t              start;  /**< Slice start offset in buffer */
        size_t              end;    /**< Slice end offset in buffer+1 */
} rd_slice_t;
```
* 用segment来初始化slice
```
int rd_slice_init_seg (rd_slice_t *slice, const rd_buf_t *rbuf,
                           const rd_segment_t *seg, size_t rof, size_t size) {
        /* Verify that \p size bytes are indeed available in the buffer. */
       // 如果需要读取的数据大小已超出buffer写入的数据量, 则返回-1
        if (unlikely(rbuf->rbuf_len < (seg->seg_absof + rof + size)))
                return -1;

        // 初始化过程
        slice->buf    = rbuf;
        slice->seg    = seg;
        slice->rof    = rof;  // 这个地方rof应该判断一下是否小于segment的seg_of
        slice->start  = seg->seg_absof + rof;
        slice->end    = slice->start + size;

        rd_assert(seg->seg_absof+rof >= slice->start &&
                  seg->seg_absof+rof <= slice->end);

        rd_assert(slice->end <= rd_buf_len(rbuf));

        return 0;
}
```
* 按指定的offset来初始化slice
```
int rd_slice_init (rd_slice_t *slice, const rd_buf_t *rbuf,
                   size_t absof, size_t size) {
        //根据offset来定位到对应buffer中的segment
        const rd_segment_t *seg = rd_buf_get_segment_at_offset(rbuf, NULL,
                                                               absof);
        if (unlikely(!seg))
                return -1;

        return rd_slice_init_seg(slice, rbuf, seg,
                                 absof - seg->seg_absof, size);
}
```
* 将整个buffer都映射到slice
```
void rd_slice_init_full (rd_slice_t *slice, const rd_buf_t *rbuf) {
        int r = rd_slice_init(slice, rbuf, 0, rd_buf_len(rbuf));
        rd_assert(r == 0);
}
```
* slice的读操作, 这个函数相当于一个slice的iterator遍历接口, 遍历这个slice对应的每一个segment
```
size_t rd_slice_reader0 (rd_slice_t *slice, const void **p, int update_pos) {
        size_t rof = slice->rof;
        size_t rlen;
        const rd_segment_t *seg;

        /* Find segment with non-zero payload */
        //  获取当前要读取的segment
        for (seg = slice->seg ;
             seg && seg->seg_absof+rof < slice->end && seg->seg_of == rof ;
              seg = TAILQ_NEXT(seg, seg_link))
                rof = 0;

        if (unlikely(!seg || seg->seg_absof+rof >= slice->end))
                return 0;

        rd_assert(seg->seg_absof+rof <= slice->end);


        *p   = (const void *)(seg->seg_p + rof);
        rlen = RD_MIN(seg->seg_of - rof, rd_slice_remains(slice));

        // 更新 slice后当前操作的segment以及rof
        if (update_pos) {
                if (slice->seg != seg) {
                        rd_assert(seg->seg_absof + rof >= slice->start &&
                                  seg->seg_absof + rof+rlen <= slice->end);
                        slice->seg  = seg;
                        slice->rof  = rlen;
                } else {
                        slice->rof += rlen;
                }
        }

        return rlen;
}
```
* 读slice到内存buffer
```
size_t rd_slice_read (rd_slice_t *slice, void *dst, size_t size) {
        size_t remains = size;
        char *d = (char *)dst; /* Possibly NULL */
        size_t rlen;
        const void *p;
        size_t orig_end = slice->end;

        if (unlikely(rd_slice_remains(slice) < size))
                return 0;

        /* Temporarily shrink slice to offset + \p size */
        slice->end = rd_slice_abs_offset(slice) + size;

       // 使用 rd_slice_reader来作iterator式的遍历
        while ((rlen = rd_slice_reader(slice, &p))) {
                rd_dassert(remains >= rlen);
                if (dst) {
                        // 内存copy
                        memcpy(d, p, rlen);
                        d       += rlen;
                }
                remains -= rlen;
        }

        rd_dassert(remains == 0);

        /* Restore original size */
        slice->end = orig_end;

        return size;
}
```
* slice的seek操作
```
int rd_slice_seek (rd_slice_t *slice, size_t offset) {
        const rd_segment_t *seg;
        size_t absof = slice->start + offset;

        if (unlikely(absof >= slice->end))
                return -1;

        // 根据offset来定位segment
        seg = rd_buf_get_segment_at_offset(slice->buf, slice->seg, absof);
        rd_assert(seg);

        // 更新slice
        slice->seg = seg;
        slice->rof = absof - seg->seg_absof;
        rd_assert(seg->seg_absof + slice->rof >= slice->start &&
                  seg->seg_absof + slice->rof <= slice->end);

        return 0;
}
```
---
### [Librdkafka源码分析-Content Table](https://www.jianshu.com/p/1a94bb09a6e6)