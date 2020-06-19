
#### 前言

`epoll`同样是linux上的IO多路复用的一种实现，内核在实现时使用的数据结构相比`select`要复杂，但原理上并不复杂，我们力求在下面的描述里抽出主干，理清思路。

`epoll`也利用了上文中介绍过的Linux中的重要数据结构 `wait queue`, 有了上面`select`的基础，其实`epoll`就没那么复杂了。

通过阅读本文 ，你除了可以了解到epoll的原理外，还可以搞清epoll存不存在惊群问题，LT和 ET模式在实现上有什么 区别，epoll和select相比有什么不同， epoll是如何处理多核并发的等等问题 。当然内容难免有疏漏之处，请大家多多指证。

#### 主要数据结构

###### eventpoll

epoll操作最重要的数据结构，封装了所有epoll操作涉及到的数据结构：

```c
struct eventpoll {
	// 用于锁定这个eventpoll数据结构，
    // 在用户空间多线程操作这个epoll结构，比如调用epoll_ctl作add, mod, del时，用户空间不需要加锁保护
    // 内核用这个mutex帮你搞定
	struct mutex mtx;

	// 等待队列，epoll_wait时如果当前没有拿到有效的事件，将当前task加入这个等待队列后作进程切换，等待被唤醒
	wait_queue_head_t wq;

	/* Wait queue used by file->poll() */
    // eventpoll对象在使用时都会对应一个struct file对象，赋值到其private_data，
    // 其本身也可以被 poll， 那也就需要一个wait queue
	wait_queue_head_t poll_wait;

	// 所有有事件触发的被监控的fd都会加入到这个列表
	struct list_head rdllist;

	/* Lock which protects rdllist and ovflist */
	rwlock_t lock;

	// 所有被监控的fd使用红黑树来存储
	struct rb_root_cached rbr;

	//  当将ready的fd复制到用户进程中，会使用上面的 lock锁锁定rdllist,
    //  此时如果有新的ready状态fd, 则临时加入到 ovflist表示的单链表中
	struct epitem *ovflist;

	// 会autosleep准备的唤醒源
	struct wakeup_source *ws;

	/* The user that created the eventpoll descriptor */
	struct user_struct *user;

    // linux下一切皆文件，epoll实例被创建时，同时会创建一个file, file的private_data
    // 指向当前这个eventpoll结构
	struct file *file;

	/* used to optimize loop detection check */
	int visited;
	struct list_head visited_list_link;

#ifdef CONFIG_NET_RX_BUSY_POLL
	/* used to track busy poll napi_id */
	unsigned int napi_id;
#endif
};
```

我们将 上面结构体中的 `poll_wait`单提出来说一下，正如注释中所说的，有了这个成员变量，那这个eventpoll对应的struct file也可以被poll，那我们也就可以将这个 epoll fd 加入到另一个epoll fd中，也就是实现了epoll的嵌套。

另外，在下面的讲解中我们暂时不涉及epoll嵌套的问题。

###### epitem

由上面的介绍我们知道每一个被 `epoll`监控的句柄都会保存在`eventpoll`内部的红黑树上（`eventpoll->rbr`），ready状态的句柄也会保存在`eventpoll`内部的一个链表上（`eventpoll->rdllist`）, 实现时会将每个句柄封装在一个结构中，即`epitem`:

```c
struct epitem {
    // 用于构建红黑树
	union {
		/* RB tree node links this structure to the eventpoll RB tree */
		struct rb_node rbn;
		/* Used to free the struct epitem */
		struct rcu_head rcu;
	};

	// 用于将当前epitem链接到eventpoll->rdllist中
	struct list_head rdllink;

	//用于将当前epitem链接到"struct eventpoll"->ovflist这个单链表中
	struct epitem *next;

	/* The file descriptor information this item refers to */
	struct epoll_filefd ffd;

	/* Number of active wait queue attached to poll operations */
	int nwait;

	/* List containing poll wait queues */
	struct list_head pwqlist;

	// 对应的eventpoll对象
	struct eventpoll *ep;

	/* List header used to link this item to the "struct file" items list */
	struct list_head fllink;

	/* wakeup_source used when EPOLLWAKEUP is set */
	struct wakeup_source __rcu *ws;

	// 需要关注的读，写事件等
	struct epoll_event event;
};
```

###### epoll_event

调用`epoll_ctl`时传入的最后一个参数，主要是用来告诉内核需要其监控哪些事件。我们先来看其定义

* 在kernel源码中的定义：

```c
  struct epoll_event {
  	__poll_t events;
  	__u64 data;
  } EPOLL_PACKED;
```

* 在glic中的定义：

```c
  typedef union epoll_data
  {
    void *ptr;
    int fd;
    uint32_t u32;
    uint64_t u64;
  } epoll_data_t;
  
  struct epoll_event
  {
    uint32_t events;	/* Epoll events */
    epoll_data_t data;	/* User data variable */
  } __EPOLL_PACKED;
```

  乍一看，为什么这两种定义不一样，这怎么调用啊？

  我们先来看下glic中的定义，它将`epoll_event.data`定义为`epoll_data_t`类型，而`epoll_data_t`被定义为`union`类型，其能表示的最大值类型为`uinit64_t`，这与kernel源码中的定义`__u64 data`是一致的，其实这个`data`成员变量部分kernel在实现时根本不会用到，它作为user data在`epoll_wait`返回时通过`epoll_event`原样返回到用户空间，声明成 `union`对使用者来说自由发挥的空间就大多了，如果使用`fd`，你可以把当前要监控的socket fd赋值给它，如果使用`void* ptr`，那你可以将任意类型指针给它......

#### 主要函数

###### epoll_create

创建一个epoll的实例，Linux里一切皆文件，这里也不例外，返回一个表示当前epoll实例的文件描述符，后续的epoll相关操作，都需要传入这个文件描述符。

其实现位于 `fs/eventpoll.c`里 `SYSCALL_DEFINE1(epoll_create, int, size)`， 具体实现 `static int do_epoll_create(int flags)`:

```c
static int do_epoll_create(int flags)
{
	int error, fd;
	struct eventpoll *ep = NULL;
	struct file *file;

	/* Check the EPOLL_* constant for consistency.  */
	BUILD_BUG_ON(EPOLL_CLOEXEC != O_CLOEXEC);

   // 目前flags只支持 EPOLL_CLOEXEC 这一种，如果传入了其他的，返回错误
	if (flags & ~EPOLL_CLOEXEC)
		return -EINVAL;
	/*
	 * Create the internal data structure ("struct eventpoll").
	 */
	error = ep_alloc(&ep);
	if (error < 0)
		return error;
	/*
	 * Creates all the items needed to setup an eventpoll file. That is,
	 * a file structure and a free file descriptor.
	 */
	fd = get_unused_fd_flags(O_RDWR | (flags & O_CLOEXEC));
	if (fd < 0) {
		error = fd;
		goto out_free_ep;
	}
	file = anon_inode_getfile("[eventpoll]", &eventpoll_fops, ep,
				 O_RDWR | (flags & O_CLOEXEC));
	if (IS_ERR(file)) {
		error = PTR_ERR(file);
		goto out_free_fd;
	}
	ep->file = file;
	fd_install(fd, file);
	return fd;

out_free_fd:
	put_unused_fd(fd);
out_free_ep:
	ep_free(ep);
	return error;
}
```

主要分以下几步：

* 校验传入参数`flags`, 目前仅支持 `EPOLL_CLOEXEC` 一种，如果是其他的，立即返回失败;

* 调用`ep_alloc`, 创建 `eventpoll`结构体;

* 在当前task的打开文件打描述符表中获取一个fd；

* 使用 `anon_inode_getfile`创建一个 匿名inode的`struct file`, 其中会使用 `file->private_data = priv`将第二步创建的`eventpoll`对象赋值给`struct file`的`private_data` 成员变量。

   关于` 匿名inode`作者也没有找到太多的资料，可以简单理解为其没有对应的`dentry`, 在目录下`ls`看不到这类文件 ，其被`close`后会自动删除，比如 使用`O_TMPFILE`选项来打开的就是这类文件;

* 将第三步中的`fd`和第四步中的`struct file`结合起来，放入当前task的打开文件描述符表中;

###### epoll_ctl

从一个fd添加到一个eventpoll中，或从中删除，或如果此fd已经在eventpoll中，可以更改其监控事件。

我们在下面的源码中添加了必要的注释：

```c
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
		struct epoll_event __user *, event)
{
	int error;
	int full_check = 0;
	struct fd f, tf;
	struct eventpoll *ep;
	struct epitem *epi;
	struct epoll_event epds;
	struct eventpoll *tep = NULL;

	error = -EFAULT;
    // ep_op_has_event() 其实就是判断当前的op不是 EPOLL_CTL_DEL操作, 
    // 如果 是EPOLL_CTL_ADD 或  EPOLL_CTL_MOD，
    //  将event由用户态复制到内核态
    // 
	if (ep_op_has_event(op) &&
	    copy_from_user(&epds, event, sizeof(struct epoll_event)))
		goto error_return;

	error = -EBADF;
	f = fdget(epfd);
	if (!f.file)
		goto error_return;

	/* Get the "struct file *" for the target file */
	tf = fdget(fd);
	if (!tf.file)
		goto error_fput;

	// 被添加的fd必须支持poll方法
	error = -EPERM;
	if (!file_can_poll(tf.file))
		goto error_tgt_fput;

	/*
	 Linux提供了autosleep的电源管理功能
	 如果当前系统支持 autosleep功能，支持休眠，
	 那么我们 允许用户传入EPOLLWAKEUP标志;
	 如果当前系统不支持这样的电源管理功能，但用户还是传入了EPOLLWAKEUP标志，
	 那么我们将此标志从flags中去掉
	*/
	if (ep_op_has_event(op))
		ep_take_care_of_epollwakeup(&epds);

	error = -EINVAL;
    // epoll不能自己监控自己
	if (f.file == tf.file || !is_file_epoll(f.file))
		goto error_tgt_fput;

	/*
	EPOLLEXCLUSIVE是为了解决某个socket有事件发生时的惊群问题
   所谓惊群，简单讲就是把一个socket fd加入到多个epoll中时，如果此socket有事件发生，
   会同时唤醒多个在此socket上等待的task
   
   目 前仅允许在EPOLL_CTL_ADD操作时传入EPOLLEXCLUSIVE标志，且传入此标志时不允许
   epoll嵌套监听
	*/
	if (ep_op_has_event(op) && (epds.events & EPOLLEXCLUSIVE)) {
		if (op == EPOLL_CTL_MOD)
			goto error_tgt_fput;
		if (op == EPOLL_CTL_ADD && (is_file_epoll(tf.file) ||
				(epds.events & ~EPOLLEXCLUSIVE_OK_BITS)))
			goto error_tgt_fput;
	}

	/*
	 * At this point it is safe to assume that the "private_data" contains
	 * our own data structure.
	 */
	ep = f.file->private_data;
    
    /*
    这里处理将一个epoll fd添加到当前epoll的嵌套情况，
    特别是要检测是否有环形epoll监听情况，类似于A监听B, B又监听A
    我们先略过
    */

	// 查看对应的epitme是否已经在红黑树上存在，即是否已经添加过
	epi = ep_find(ep, tf.file, fd);

	error = -EINVAL;
	switch (op) {
	case EPOLL_CTL_ADD:
		if (!epi) {
			epds.events |= EPOLLERR | EPOLLHUP;
            // 将当前fd加入红黑树，我们在下面重点讲
			error = ep_insert(ep, &epds, tf.file, fd, full_check);
		} else
			error = -EEXIST;
		if (full_check)
			clear_tfile_check_list();
		break;
	case EPOLL_CTL_DEL:
		if (epi)
			error = ep_remove(ep, epi);
		else
			error = -ENOENT;
		break;
	case EPOLL_CTL_MOD:
		if (epi) {
			if (!(epi->event.events & EPOLLEXCLUSIVE)) {
				epds.events |= EPOLLERR | EPOLLHUP;
				error = ep_modify(ep, epi, &epds);
			}
		} else
			error = -ENOENT;
		break;
	}
    ......
	return error;
}
```

这个函数主要作以下几件事：

* 先将epoll_event(上面已有介绍，保存着需要监控的事件)从用户空间复制到内核空间。

  我们看来针对某个socket, 这种用户空间到内核空间的复制只需一次，不像`select`，每次调用都要复制；

* 先由传入的epoll fd和被监听的socket fd获取到其对应的文件句柄 `struct file`，针对文件句柄和传入的flags作边界条件检测；

* 针对epoll嵌套用法，作单独检测，检测是否有环形epoll监听情况，类似于A监听B, B又监听A， 这部分我们先略过；

* 针对 `EPOLL_CTL_ADD` `EPOLL_CTL_DEL` `EPOLL_CTL_MOD`分别作处理。

###### ep_insert

这个函数是真正将待监听的fd加入到epoll中去。下面我们将这个函数的实现拆解，分段来看一下其是如何实现的。

* 作max_user_watches检验

```c
  	user_watches = atomic_long_read(&ep->user->epoll_watches);
  	if (unlikely(user_watches >= max_user_watches))
  		return -ENOSPC;
```

  内核对系统中所有（是所有，所有使用了epoll的进程）使用`epoll`监听fd所消耗的内存作了限制， 且这个限制是针对当前linux user id的。32位系统为了监控注册的每个文件描述符大概占90字节，64位系统上占160字节。

  可以通过 ` /proc/sys/fs/epoll/max_user_watches`来查看和设置 。

  默认情况下每个用户下`epoll`为注册文件描述符可用的内存是内核可使用内存的1/25。

* 初始化`epitem`

  这个`epitem`前面说过，它会被挂在epoll的红黑树上。

```c
  	if (!(epi = kmem_cache_alloc(epi_cache, GFP_KERNEL)))
  		return -ENOMEM;
  
  	/* Item initialization follow here ... */
  	INIT_LIST_HEAD(&epi->rdllink);
  	INIT_LIST_HEAD(&epi->fllink);
  	INIT_LIST_HEAD(&epi->pwqlist);
  	epi->ep = ep;
  	ep_set_ffd(&epi->ffd, tfile, fd);
  	epi->event = *event;
  	epi->nwait = 0;
  	epi->next = EP_UNACTIVE_PTR;
  	if (epi->event.events & EPOLLWAKEUP) {
  		error = ep_create_wakeup_source(epi);
  		if (error)
  			goto error_create_wakeup_source;
  	} else {
  		RCU_INIT_POINTER(epi->ws, NULL);
  	}
```

  a. 这个epitem里会保存监控的fd及其事件，所属的eventpoll等；

  b. 如果events里设置了EPOLLWAKEUP, 还需要为autosleep创建一个唤醒源 `ep_create_wakeup_source`。

* 获取当前被监听的fd上是否有感 兴趣的事件发生，同时生成新的`eppoll_entry`对象并添加到被监听的socket fd的等待队列中

```c
      epq.epi = epi;
  	init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);
  	revents = ep_item_poll(epi, &epq.pt, 1);
```

  下面是 `ep_item_poll`的实现：

```c
  static __poll_t ep_item_poll(const struct epitem *epi, poll_table *pt,
  				 int depth)
  {
  	struct eventpoll *ep;
  	bool locked;
  
  	pt->_key = epi->event.events;
  	if (!is_file_epoll(epi->ffd.file))
  		return vfs_poll(epi->ffd.file, pt) & epi->event.events;
  }
```

  如果你读过上面的`select`分析部分，就会看到一个熟悉的身影 `vfs_poll`,  它会调用 `ep_ptable_queue_proc`将当前被监听的socket fd加入到等待队列中：

```c
  static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,
  				 poll_table *pt)
  {
  	struct epitem *epi = ep_item_from_epqueue(pt);
  	struct eppoll_entry *pwq;
  
  	if (epi->nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {
  		init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);
  		pwq->whead = whead;
  		pwq->base = epi;
  		if (epi->event.events & EPOLLEXCLUSIVE)
  			add_wait_queue_exclusive(whead, &pwq->wait);
  		else
  			add_wait_queue(whead, &pwq->wait);
  		list_add_tail(&pwq->llink, &epi->pwqlist);
  		epi->nwait++;
  	} else {
  		/* We have to signal that an error occurred */
  		epi->nwait = -1;
  	}
  }
```

  这里有两点比较重要：

  a. `init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);`如果这个被监听的socket上有事件发生，这个回调       `ep_poll_callback`将被调用, 我们后面会讲这个回调里作了哪些事情, 这个回调很重要;

  b. 如果设置了 `EPOLLEXCLUSIVE`，  将使用`add_wait_queue_exclusive`添加到等待队列。意思是说，如果一个socket fd被添加到了多个epoll中进行监控，设置了这个参数后，这个fd上有事件发生时，只会唤醒被添加到的第一个epoll里，避免惊群。

* 添加到 epoll的红黑树上

```c
  ep_rbtree_insert(ep, epi);
```

* 如果上面调用 `ep_item_poll`时，立即返回了准备好的事件，我们这里要作唤醒的操作

```c
  	if (revents && !ep_is_linked(epi)) {
  		list_add_tail(&epi->rdllink, &ep->rdllist);
  		ep_pm_stay_awake(epi);
  
  		/* Notify waiting tasks that events are available */
  		if (waitqueue_active(&ep->wq))
  			wake_up(&ep->wq);
  		if (waitqueue_active(&ep->poll_wait))
  			pwake++;
  	}
  
      .....
       	if (pwake)
  		ep_poll_safewake(&ep->poll_wait);
```

  a. 将当前 epi加入到eventpoll的rdllist中；

  b. 如果当前eventpoll处于wait状态，就唤醒它;

  c.  如果当前的eventpoll被嵌套地加入到了另外的poll中，且处于wait状态，就唤醒它。

###### `ep_poll_callback`

被监听的socket fd上有事件发生时，这个回调被触发， 然后唤醒`epoll_wait`被调用时加入到eventpoll等待队列中的task，下面会放张图来解释其功能。

###### `ep_events_available`

我们首先来看一下函数`ep_events_available`,它的功能是检测当前epoll上是否已经收集了有效的事件：

```c
static inline int ep_events_available(struct eventpoll *ep)
{
	return !list_empty_careful(&ep->rdllist) ||
		READ_ONCE(ep->ovflist) != EP_UNACTIVE_PTR;
}
```

按这个逻辑只有rdllist不为空或者ovflist != EP_UNACTIVE_PTR，那么就有有效的事件，前一个条件好理解，ovflist这个我们先在这里埋个坑，后面我们来填它～

###### `ep_poll`

这个函数是`epoll_wait`在内核里的具体实现。我们把它的实现分解来看。

* 准备好超时时间

```c
  if (timeout > 0) {
  		struct timespec64 end_time = ep_set_mstimeout(timeout);
  
  		slack = select_estimate_accuracy(&end_time);
  		to = &expires;
  		*to = timespec64_to_ktime(end_time);
  	} else if (timeout == 0) {
  		timed_out = 1;
  
  		write_lock_irq(&ep->lock);
  		eavail = ep_events_available(ep);
  		write_unlock_irq(&ep->lock);
  
  		goto send_events;
  	}
```

  a. 如果用户设置了超时时间， 作相应的初始化;

  b.  如果timeout == 0, 表时此次调用立即返回， 此时首先获取当前是否已有有效的事件ready, 然后goto 到send_events, 这部分是将有效的events复制到用户空间，我们后面会详述。

* 将当前task加入到此eventpoll的等待队列中

```c
  if (!waiter) {
  		waiter = true;
  		init_waitqueue_entry(&wait, current);
  
  		spin_lock_irq(&ep->wq.lock);
  		__add_wait_queue_exclusive(&ep->wq, &wait);
  		spin_unlock_irq(&ep->wq.lock);
  	}
```

  我们前面在`select`部分已经介绍过wait queue, 这里就是将当前task加入到eventpoll的等待队列，接下来当前task将会被调度走，然后等待从eventpoll的等待队列中被唤醒。这里用了`__add_wait_queue_exclusive`， 是说针对同一个`eventpoll`, 可能在不同的进程(线程)调用`epoll_wait`,  此时eventpoll的等待队列里将会有多个task, 为避免惊群，我们每次只唤醒一个task。

* 无限循环体

```c
  for (;;) {
  		set_current_state(TASK_INTERRUPTIBLE);
  
  		if (fatal_signal_pending(current)) {
  			res = -EINTR;
  			break;
  		}
  
  		eavail = ep_events_available(ep);
  		if (eavail)
  			break;
  		if (signal_pending(current)) {
  			res = -EINTR;
  			break;
  		}
  
  		if (!schedule_hrtimeout_range(to, slack, HRTIMER_MODE_ABS)) {
  			timed_out = 1;
  			break;
  		}
  	}
```

  这个无限循环体退出的条件：

  a. 有signal发生，被中断会退出;

  b.  有ready的事件，会退出;

  c.  用户设置的超时时间到达，会退出;

  否则当前 task将被 `schedule_hrtimeout_range`调度走。

* 有ready的事件复制到用户空间

```c
  	if (!res && eavail &&
  	    !(res = ep_send_events(ep, events, maxevents)) && !timed_out)
  		goto fetch_events;
```

* `ep_poll`和`ep_poll_callback`的处理有些复杂，我下面用张图来说明一下：

![epoll.png](https://upload-images.jianshu.io/upload_images/2020390-1ff5240b545434f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


  几点说明如下：

  1.  实际中可能同时有多个socket有事件到来，此时`ep_poll_callback`会并发被调用，因此将epi添中到eventpoll->rdllik时，均采用原子操作;

  2. `ep_scan_ready_list`中一旦开始向用户空间复制events, eventpoll->rdllink就不能再有新的添加，此时如果`ep_poll_callback`被调用，当前的epi会被添加到eventpoll->ovflist中， ovflist是个单链表，这个添加操作很有意思，每次新的epi都被原子添加到链接头：

```c
     static inline bool chain_epi_lockless(struct epitem *epi)
     {
     	struct eventpoll *ep = epi->ep;
     
     	/* Check that the same epi has not been just chained from another CPU */
     	if (cmpxchg(&epi->next, EP_UNACTIVE_PTR, NULL) != EP_UNACTIVE_PTR)
     		return false;
     
     	/* Atomically exchange tail */
     	epi->next = xchg(&ep->ovflist, epi);
     
     	return true;
     }
```

  3. `ep_send_events_proc`才是真正实现将events复制到用户空间。

     虽然当socket fd有事件到来时，会通过`ep_poll_callback`来唤醒epoll_wait所在的task, 后者遍历rdllist即可，但在遍历时，还是通过`ep_item_poll`（内部会调用vs_poll, 最终调用到tcp_poll）来获取关注的事件是否发生，所有poll机制很重要；

  4. 对于水平触发方式，在首次调用`ep_item_poll`后，会再次将这个`epi`加入到eventpoll->rdllist这个就绪列表中，这会导致两种情况出现：

     a.  如果针对同一个eventpoll同时调用了多个 epoll_wait, 此时另一个调用epoll_wait的task将被唤醒，这不能被称之为`epoll_wait`的惊群，反而是并发处理的体现；

     b. 如果只有一个epoll_wait, 那下次这个epoll_wait再次被调用时，不会进入到上面的`无限循化`逻辑，也不会被调度走，而是直接又一次进入到`ep_send_events`中，直到在这个socket fd上poll不到关注的事情，它就不会再被加入到rdllist中。你可以将这个水平触发方式理解成是完全轮询的一种实现；

     聪明的你读到这里一定会发现对于水平触发，即使是socket fd上已经没有关注的事件发生了，它还是要多用一次poll来确认，这是一处性能损失的点，但监听的socket少的话这也不是什么大问题。

#### 总结

这里先讲上点上面没有提及的内容

* epoll模型中ep_poll执行时如果当前没有有效的events，当前task会被调度走，后续有socket fd有事件发生，ep_poll_callback被调用，将当前的socket fd 添加到rdllist中，再唤醒前面的task, 然后ep_poll再一次被调度执行，锁定住rdllist后开始向用户空间复制，由次可以看出来每次epoll_wait返回的events就是从第一次ep_poll_callback调用执行唤醒到ep_poll所有task被真正唤醒开始执行这段时间内，所收集中的socket fd。如果同时有大量的socket fd是活跃状态，那么这里可能需要多次调用epoll_wait，效率上是个问题；

  