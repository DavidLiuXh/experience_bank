* Librdkafka用纯C写成,作者在C API基础上作了C++的简单封装;
* 说到C, 自然里面离不开大量的指针操作, 内存操作, 引用计数等等, 作者一一为我们作了实现;
* 基础数据结构里面也说到了很多,比如 链表, 各种队列, 二叉树等等, 接下来我们会一一介绍.
---
###### Queue
* 所在文件: src/queue.h和src/rdsysqueue.h
* `queue.h`是个很古老的实现了, 作者这里用的是NetBsd里的实现, 还有其他很多版本, 实现都大同小异, 里面指针的运用真是值得我们好好学习;
* 网上关于这个的讲解有很多, 我们这里只是简单过一下用得最多的Tail queue;
* 头指针定义 `#define TAILQ_HEAD(name, type)	_TAILQ_HEAD(name, struct type,)
`:
```
#define	_TAILQ_HEAD(name, type, qual)					\
struct name {								\
	qual type *tqh_first;		/* first element */		\
	qual type *qual *tqh_last;	/* addr of last next element */	\
}
```
两个元素:
`tqh_first`: 指向队列的第一个成员;
`tqh_last`: 存的是队列里的最后一个元素的 next指针的变量地址, 这个二级指针太有用了,我们后边会再讲到;
* 队列的entry: `#define TAILQ_ENTRY(type)	_TAILQ_ENTRY(struct type,)
` 实际是用最少侵入式的方式实现了一个类似于C++的模板的机制, 定义中的`type`就是队列里元素的类型, 可以是任意struct类型, 这个`_TAILQ_ENTRY(type, qual)`放在这个struct类型的定义里,是其的一个成员, 然后各个元素通过这个`TAILQ_ENTRY`成员彼此串联起来, 松耦合在一起
```
#define	_TAILQ_ENTRY(type, qual)					\
struct {								\
	qual type *tqe_next;		/* next element */		\
	qual type *qual *tqe_prev;	/* address of previous next element */\
}
#define TAILQ_ENTRY(type)	_TAILQ_ENTRY(struct type,)
```
两个元素:
`tqe_next`: 指向队列的下一个成员;
`tqe_prev`: 存的是前一个元素的 next指针的变量地址, 这个二级指针太有用了,我们后边会再讲到;
* 获队列的最后一个元素`#define	TAILQ_LAST(head, headname)`:
```
(*(((struct headname *)((head)->tqh_last))->tqh_last))
```
要理解这个其实关键一点是上面定义的`TAILQ_HEAD`和`TAILQ_ENTRY`在结构和内存布局上是一样一样的.
`(head)->tqh_last)`是最后一个元素的next指针的地址, 因为`TAILQ_ENTRY(type)`这个定义是没有类型名的,我们不能直接cast成 `TAILQ_ENTRY(type)`类型, 所有就只能cast成`(struct headname *)`, 所有((struct headname *)((head)->tqh_last))也就是最后一个元素的地址,(((struct headname *)((head)->tqh_last))->tqh_last)就是最后一个元素的`tqe_prev`值, 这个`tqe_prev`指向的是它前一个元素的`next`的地址, 解引用后自然就指向队列最后一个元素自己了,  有点绕~~~
* 获取当前元素的前一个元素 `#define	TAILQ_PREV(elm, headname, field)`
```
(*(((struct headname *)((elm)->field.tqe_prev))->tqh_last))
```
有了上一个取最后元素的经验,这个理解起来就容易多啦, 这里就不讲啦~~~
* 其他一些操作, 头插,尾插, 前插,后插, 遍历, 反向遍历, 删除都比较简单了, 主要也都是指针操作;
###### rd_list_t
* 是一个append-only的轻量级数组链表, 支持固定大小和按需自动增长两种方式且支持sort;
* 所在文件: src/rdlist.h 和 src/rdlist.c
* 定义:
```
typedef struct rd_list_s {
        int    rl_size;  // 目前list的容易
        int    rl_cnt;  // 当前list里已放入的元素个数
        void **rl_elems; // 存储元素的内存, 可以看成是一个void*类型的数组
	    void (*rl_free_cb) (void *); // 存储的元素的释放函数
	    int    rl_flags; 
        #define RD_LIST_F_ALLOCATED  0x1  /* The rd_list_t is allocated,
				   * will be free on destroy() */
        #define RD_LIST_F_SORTED     0x2  /* Set by sort(), cleared by any mutations.
				   * When this flag is set bsearch() is used
				   * by find(), otherwise a linear search. */
        #define RD_LIST_F_FIXED_SIZE 0x4  /* Assert on grow */
        #define RD_LIST_F_UNIQUE     0x8  /* Don't allow duplicates:
                                   * ONLY ENFORCED BY CALLER. */
} rd_list_t;
```
* 创建list: `rd_list_t *rd_list_new (int initial_size, void (*free_cb) (void *))`
```
rd_list_t *rd_list_new (int initial_size, void (*free_cb) (void *)) {
	rd_list_t *rl = malloc(sizeof(*rl));
	rd_list_init(rl, initial_size, free_cb);
	rl->rl_flags |= RD_LIST_F_ALLOCATED;
	return rl;
}
```
* 容量自增: `void rd_list_grow (rd_list_t *rl, size_t size)`
主要是调用`rd_realloc`(realloc)
```
void rd_list_grow (rd_list_t *rl, size_t size) {
        rd_assert(!(rl->rl_flags & RD_LIST_F_FIXED_SIZE)); // 不适用于固定大小类型的list
        rl->rl_size += (int)size;
        if (unlikely(rl->rl_size == 0))
                return; /* avoid zero allocations */
        rl->rl_elems = rd_realloc(rl->rl_elems,
                                  sizeof(*rl->rl_elems) * rl->rl_size);
}
```
* 固定容量的list的创建: `void rd_list_prealloc_elems (rd_list_t *rl, size_t elemsize, size_t size) 
`
实际上连每个元素的大小也是固定的
```
oid rd_list_prealloc_elems (rd_list_t *rl, size_t elemsize, size_t size) {
	size_t allocsize;
	char *p;
	size_t i;

	rd_assert(!rl->rl_elems);

	/* Allocation layout:
	 *   void *ptrs[cnt];
	 *   elems[elemsize][cnt];
	 */

	allocsize = (sizeof(void *) * size) + (elemsize * size); //rl_elems数组本身的大小(元素类型是void*) + 所有元素的大小
	rl->rl_elems = rd_malloc(allocsize);

	/* p points to first element's memory. */
	p = (char *)&rl->rl_elems[size];

	/* Pointer -> elem mapping */
	for (i = 0 ; i < size ; i++, p += elemsize)
		rl->rl_elems[i] = p; //数组元素赋值

	rl->rl_size = (int)size;
	rl->rl_cnt = 0;
	rl->rl_flags |= RD_LIST_F_FIXED_SIZE; // 标识为固定大小的list
}
```
* 添加新元素 `void *rd_list_add (rd_list_t *rl, void *elem)`:
```
void *rd_list_add (rd_list_t *rl, void *elem) {
        if (rl->rl_cnt == rl->rl_size)
                rd_list_grow(rl, rl->rl_size ? rl->rl_size * 2 : 16); //容量不够先自增
	rl->rl_flags &= ~RD_LIST_F_SORTED;  //新增后原有的排序失效
	if (elem)
		rl->rl_elems[rl->rl_cnt] = elem;
	return rl->rl_elems[rl->rl_cnt++];  //当前已有元素数 + 1
}
```
* 按索引(相当于数据的下标)移除元素:
```
static void rd_list_remove0 (rd_list_t *rl, int idx) {
        rd_assert(idx < rl->rl_cnt);

        if (idx + 1 < rl->rl_cnt) //如果idx指向的是最后一个元素,不用mm了, rl->rl_cnt--就好
                memmove(&rl->rl_elems[idx],
                        &rl->rl_elems[idx+1],
                        sizeof(*rl->rl_elems) * (rl->rl_cnt - (idx+1))); // 实际就是用memmove作内存移动
        rl->rl_cnt--;
}
```
* 排序用的`qsort`快排
```
void rd_list_sort (rd_list_t *rl, int (*cmp) (const void *, const void *)) {
	rd_list_cmp_curr = cmp;
        qsort(rl->rl_elems, rl->rl_cnt, sizeof(*rl->rl_elems),
	      rd_list_cmp_trampoline);
	rl->rl_flags |= RD_LIST_F_SORTED;
}
```
* 查找:
```
void *rd_list_find (const rd_list_t *rl, const void *match,
                    int (*cmp) (const void *, const void *)) {
        int i;
        const void *elem;

       //如果排过序就用二分查找
	if (rl->rl_flags & RD_LIST_F_SORTED) {
		void **r;
		rd_list_cmp_curr = cmp;
		r = bsearch(&match/*ptrptr to match elems*/,
			    rl->rl_elems, rl->rl_cnt,
			    sizeof(*rl->rl_elems), rd_list_cmp_trampoline);
		return r ? *r : NULL;
	}

       没排过就编历, 是不是可以先qsort一下呢?
        RD_LIST_FOREACH(elem, rl, i) {
                if (!cmp(match, elem))
                        return (void *)elem;
        }

        return NULL;
}
```
---
### [Librdkafka源码分析-Content Table](https://www.jianshu.com/p/1a94bb09a6e6)