#### 前言

通过阅读本文，帮你理清`select`的来龙去脉， 你可以从中了解到：

* 我们常说的`select`的1024限制指的是什么 ？怎么会有这样的限制？
* 都说`select`效率不高，是这样吗？为什么 ？
* `select`使用中有坑吗？

注：本文的所有内容均指针对 Linux Kernel, 当前使用的源码版本是 5.3.0

#### 原型

```
int select (int __nfds, fd_set *__restrict __readfds,
		   fd_set *__restrict __writefds,
		   fd_set *__restrict __exceptfds,
		   struct timeval *__restrict __timeout);
```

 如你所知，`select`是IO多种复用的一种实现，它将需要监控的fd分为读，写，异常三类，使用`fd_set`表示，当其返回时要么是超时，要么是有至少一种读，写或异常事件发生。

#### 相关数据结构

##### FD_SET

`FD_SET`是`select`最重要的数据结构了，其在内核中的定义如下：

```
typedef __kernel_fd_set		fd_set;
#undef __FD_SETSIZE
#define __FD_SETSIZE	1024

typedef struct {
	unsigned long fds_bits[__FD_SETSIZE / (8 * sizeof(long))];
} __kernel_fd_set;
```

我们来简化下，`fd_set`是一个`struct`, 其内部只有一个 由16个元素组成的`unsigned long`数组，这个数组一共可以表示`16 × 64 = 1024`位， 每一位用来表示一个 `fd`, 这也就是 `select`针对读，定或异常每一类最多只能有 `1024`个fd 限制的由来。

#### 相关宏

下面这些宏定义在内核代码`fs/select.c`中

* `FDS_BITPERLONG`: 返回每个`long`有多少位，通常是64bits

  ```
   #define FDS_BITPERLONG	(8*sizeof(long))
  ```

* `FDS_LONGS(nr)`: 获取 nr 个fd 需要用几个`long`来表示

  ```
 #define FDS_LONGS(nr)	(((nr)+FDS_BITPERLONG-1)/FDS_BITPERLONG)
  ```

* `FD_BYTES(nr)`: 获取 nr 个fd 需要用 多少个字节来表示

  ```
  #define FDS_BYTES(nr)	(FDS_LONGS(nr)*sizeof(long))
  ```

下面这些宏可以在gcc源码中找到

* `FD_ZERO`: 初始化一个`fd_set`

```
  #define __FD_ZERO(s) \
    do {									      \
      unsigned int __i;							      \
      fd_set *__arr = (s);						      \
      for (__i = 0; __i < sizeof (fd_set) / sizeof (__fd_mask); ++__i)	      \
        __FDS_BITS (__arr)[__i] = 0;					      \
    } while (0)
```

  将上面所说的由16个元素组成的`unsigned long`数组每一个元素都设为 0;

* `__FD_SET(d, s)`: 将一个fd 赋值到 一个 `fd_set`

```
  #define __FD_SET(d, s) \
    ((void) (__FDS_BITS (s)[__FD_ELT(d)] |= __FD_MASK(d)))
```

  分三步：

  a.  `__FD_ELT(d)`: 确定赋值到数组的哪一个元素

```
  #define	__FD_ELT(d) \
    __extension__								    \
    ({ long int __d = (d);						    \
       (__builtin_constant_p (__d)					    \
        ? (0 <= __d && __d < __FD_SETSIZE					    \
  	 ? (__d / __NFDBITS)						    \
  	 : __fdelt_warn (__d))						    \
       : __fdelt_chk (__d)); })
```


  其中 `#define __NFDBITS	(8 * (int) sizeof (__fd_mask))` ， 即`__NFDBITS =  64`

  这里实现使用了`__builtin_constant_p`针对常量作了优化，我也没有太理解常量与非常量实现方案有什么不同，我们暂时忽略这个细节看本质。

  本质就是 一个 `unsigned long`有64位，直接 `__d / __NFDBITS`取模就可以确定用数组的哪一个元素了；

  b. `__FD_MASK(d)`: 确定赋值到一个 `unsigned long`的哪一位

  ```
  #define	__FD_MASK(d)	((__fd_mask) (1UL << ((d) % __NFDBITS)))
  ```

  直接 `（d) % __NFDBITS)`取余后作为 1 左移的位数即可

  c. `|=` ：用 位或 赋值即可;

#### 在内核中的实现

##### 调用层级

![select.png](https://upload-images.jianshu.io/upload_images/2020390-b9752597452106e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##### 系统调用入口位置

位于`fs/select.c`中

```
SYSCALL_DEFINE5(select, int, n, fd_set __user *, inp, fd_set __user *, outp,
		fd_set __user *, exp, struct timeval __user *, tvp)
{
	return kern_select(n, inp, outp, exp, tvp);
}
```

##### 入口函数 `kern_select`

```
static int kern_select(int n, fd_set __user *inp, fd_set __user *outp,
		       fd_set __user *exp, struct timeval __user *tvp)
{
	struct timespec64 end_time, *to = NULL;
	struct timeval tv;
	int ret;

	if (tvp) {
		if (copy_from_user(&tv, tvp, sizeof(tv)))
			return -EFAULT;

		to = &end_time;
		if (poll_select_set_timeout(to,
				tv.tv_sec + (tv.tv_usec / USEC_PER_SEC),
				(tv.tv_usec % USEC_PER_SEC) * NSEC_PER_USEC))
			return -EINVAL;
	}

	ret = core_sys_select(n, inp, outp, exp, to);
	return poll_select_finish(&end_time, tvp, PT_TIMEVAL, ret);
}
```

作三件事：

a. 如果设置了超时，首先准备时间戳 `timespec64`;

b. 调用 `core_sys_select`，这个是具体的实现，我们下面会重点介绍

c. `poll_select_finish`：作的主要工作就是更新用户调用`select`时传进来的 超时参数`tvp`,我列一下关键代码：

```
	ktime_get_ts64(&rts);
	rts = timespec64_sub(*end_time, rts);
	if (rts.tv_sec < 0)
		rts.tv_sec = rts.tv_nsec = 0;

...
	struct timeval rtv;

			if (sizeof(rtv) > sizeof(rtv.tv_sec) + sizeof(rtv.tv_usec))
				memset(&rtv, 0, sizeof(rtv));
			rtv.tv_sec = rts.tv_sec;
			rtv.tv_usec = rts.tv_nsec / NSEC_PER_USEC;
			if (!copy_to_user(p, &rtv, sizeof(rtv)))
				return ret;
```

可以看到先获取当前的时间戳，然后通过`timespec64_sub`和传入的时间戳（接中传入的是超时时间，实现时会转化为时间戳）求出差值，将此差值传回给用户，即返回了剩余的超时时间。所以这个地方是个小陷阱，用户在调用`select`时，需要每次重新初始化这个超时时间。

##### 通过 core_sys_select  实现

这个函数主要功能是在实现真正的select功能前，准备好 `fd_set` ，即从用户空间将所需的三类 `fd_set` 复制到内核空间。从下面的代码中你会看到对于每次的 `select`系统调用，都需要从用户空间将所需的三类 `fd_set` 复制到内核空间，这里存在性能上的损耗。

```
int core_sys_select(int n, fd_set __user *inp, fd_set __user *outp,
			   fd_set __user *exp, struct timespec64 *end_time)
{
	fd_set_bits fds;
	void *bits;
	int ret, max_fds;
	size_t size, alloc_size;
	struct fdtable *fdt;
	/* Allocate small arguments on the stack to save memory and be faster */
	long stack_fds[SELECT_STACK_ALLOC/sizeof(long)];

	ret = -EINVAL;
	if (n < 0)
		goto out_nofds;

	/* max_fds can increase, so grab it once to avoid race */
	rcu_read_lock();
	fdt = files_fdtable(current->files);
	max_fds = fdt->max_fds;
	rcu_read_unlock();
	if (n > max_fds)
		n = max_fds;

	/*
	 * We need 6 bitmaps (in/out/ex for both incoming and outgoing),
	 * since we used fdset we need to allocate memory in units of
	 * long-words. 
	 */
	size = FDS_BYTES(n);
	bits = stack_fds;
	if (size > sizeof(stack_fds) / 6) {
		/* Not enough space in on-stack array; must use kmalloc */
		ret = -ENOMEM;
		if (size > (SIZE_MAX / 6))
			goto out_nofds;

		alloc_size = 6 * size;
		bits = kvmalloc(alloc_size, GFP_KERNEL);
		if (!bits)
			goto out_nofds;
	}
	fds.in      = bits;
	fds.out     = bits +   size;
	fds.ex      = bits + 2*size;
	fds.res_in  = bits + 3*size;
	fds.res_out = bits + 4*size;
	fds.res_ex  = bits + 5*size;

	if ((ret = get_fd_set(n, inp, fds.in)) ||
	    (ret = get_fd_set(n, outp, fds.out)) ||
	    (ret = get_fd_set(n, exp, fds.ex)))
		goto out;
	zero_fd_set(n, fds.res_in);
	zero_fd_set(n, fds.res_out);
	zero_fd_set(n, fds.res_ex);

	ret = do_select(n, &fds, end_time);

	if (ret < 0)
		goto out;
	if (!ret) {
		ret = -ERESTARTNOHAND;
		if (signal_pending(current))
			goto out;
		ret = 0;
	}

	if (set_fd_set(n, inp, fds.res_in) ||
	    set_fd_set(n, outp, fds.res_out) ||
	    set_fd_set(n, exp, fds.res_ex))
		ret = -EFAULT;

out:
	if (bits != stack_fds)
		kvfree(bits);
out_nofds:
	return ret;
}
```

代码中的注释很清晰，我们这里简单过一下：

分五步：

* 规范化`select`系统调用传入的第一个参数 n

```
  	rcu_read_lock();
  	fdt = files_fdtable(current->files);
  	max_fds = fdt->max_fds;
  	rcu_read_unlock();
  	if (n > max_fds)
  		n = max_fds;
```

  这个n是三类不同的`fd_set`中所包括的fd数值的最大值 + 1, linux task打开句柄从0开始，不加1的话可能会少监控fd.

  用户在使用时可以有个偷懒的作法，就是将这个n设置 为 `FD_SETSIZE`，通常 是1024,  这将监控的范围扩大到了上限，但实际上远没有这么多fd需要监控，浪费资源。

  linux man中的解释如下：

  > nfds should be set to the highest-numbered file descriptor in any of the three sets, plus 1.  The  indicated file descriptors in each set are checked, up to this limit (but see BUGS).

*  计算内核空间所需要的`fd_set`的空间, 内核态需要三个`fd_set`来容纳用户态传递过来的参数，还需要三个`fd_set`来容纳`select`调用返回后生成的三灰`fd_set`, 即一共是6个`fd_set`

```
     long stack_fds[SELECT_STACK_ALLOC/sizeof(long)];
     size = FDS_BYTES(n);
  	bits = stack_fds;
  	if (size > sizeof(stack_fds) / 6) {
  		/* Not enough space in on-stack array; must use kmalloc */
  		ret = -ENOMEM;
  		if (size > (SIZE_MAX / 6))
  			goto out_nofds;
  
  		alloc_size = 6 * size;
  		bits = kvmalloc(alloc_size, GFP_KERNEL);
  		if (!bits)
  			goto out_nofds;
  	}
  	fds.in      = bits;
  	fds.out     = bits +   size;
  	fds.ex      = bits + 2*size;
  	fds.res_in  = bits + 3*size;
  	fds.res_out = bits + 4*size;
  	fds.res_ex  = bits + 5*size;
```

  这里有个小技巧，先从内核栈上分配空间，如果不够用，才使用 `kvmalloc`分配。

  通过 `size = FDS_BYTES(n);`计算出单一一种`fd_set`所需字节数，

  再能过 `alloc_size = 6 * size;` 即可计算出所需的全部字节数。

* 初始化用作参数的和用作返回值的两类`fd_set`

  ```
  if ((ret = get_fd_set(n, inp, fds.in)) ||
  	    (ret = get_fd_set(n, outp, fds.out)) ||
  	    (ret = get_fd_set(n, exp, fds.ex)))
  		goto out;
  	zero_fd_set(n, fds.res_in);
  	zero_fd_set(n, fds.res_out);
  	zero_fd_set(n, fds.res_ex);
  ```

  主要使用 `copy_from_user`和 `memset`来实现。

* 真正实现部分 `do_select`， 我们在下面详讲

* 返回结果复制回用户空间

```
  if (set_fd_set(n, inp, fds.res_in) ||
  	    set_fd_set(n, outp, fds.res_out) ||
  	    set_fd_set(n, exp, fds.res_ex))
  		ret = -EFAULT;
```

​        这里又多了一次内核空间到用户空间的copy, 而且我们看到返回值也是用`fd_set`结构来表示，这意味着我们在用户空      间处理里也需要遍历每一位。

##### 精华所在 `do_select`

###### wait queue

这里用到了Linux里一个很重要的数据结构 `wait queue`, 我们暂不打算展开来讲，先简单来说下其用法，比如我们在进程中read时经常要等待数据准备好，我们用伪码来写些流程：

```
// Read 代码
for (true) {
    if 数据准备好 {
        拷贝数据到用户空间buffer
        return
    } else {
       创建一个 wait_queue_entry_t  wait_entry;
       wait_entry.func = 自定义函数，被唤醒时会调用
       wait_entry.private = 自定义的数据结构

       将此 wait_entry 加入要读取数据的 设备的等待队列

       set_current_state(TASK_INTERRUPTIBLE) // 将当前进程状态设置为 TASK_INTERRUPTIBLE

       schedule() // 将当前进程调度走，进行进程切换
    }
}

// 设备驱动端代码
if 设备有数据可读 {
   for ( 遍历其wait_queue ) {
      唤醒 每一个 wait_queue_entry
      调用 wait_entry.func {
           将上面读取进程状态设置为 TASK_RUNNING，并加入CPU核的运行队列，被再次调度后，将读取到数据
      }
   }
}
```

###### `do_select`源码走读

* 获取当前三类`fd_set`中最大的fd

```
      rcu_read_lock();
  	retval = max_select_fd(n, fds);
  	rcu_read_unlock();
  	
 	n = retval; 
```

  上面 `n = retval`中的 `n`, 即为三类`fd_set`中最大的fd,  也是下面要介绍的循环体的上限

* 初始化用作wait queue中的  wait entry 的private数据结构

  ```
  poll_initwait(&table);
  ```

* 核心循环体

  我们讲注释写在这个代码块里

```
  // 最外面是一个无限循环，它只有在poll到有效的事件，或者超时，或者有中断发生时，才会退出
  for (;;) {
  		unsigned long *rinp, *routp, *rexp, *inp, *outp, *exp;
  		bool can_busy_loop = false;
  
         // 首先获取需要监控的三类fd_set, 其实是 unsigned long 数组
  		inp = fds->in; outp = fds->out; exp = fds->ex;
         // 初始化用于保存返回值的三类 fd_set对应的unsigned long 数组
  		rinp = fds->res_in; routp = fds->res_out; rexp = fds->res_ex;
  
          // 开始循环遍历覆盖的所有fd, 以上面得到的 n 上限
  		for (i = 0; i < n; ++rinp, ++routp, ++rexp) {
  			unsigned long in, out, ex, all_bits, bit = 1, j;
  			unsigned long res_in = 0, res_out = 0, res_ex = 0;
  			__poll_t mask;
              
  			in = *inp++; out = *outp++; ex = *exp++;
  			all_bits = in | out | ex;
  			if (all_bits == 0) {
                  //  如果走到这里，说明在三类fd_set的数组中，与当前下标对应的三个unsigned long的每一位均为0, 即当前
                  // 不存在任何监控的fd, 开始下一次循环
  				i += BITS_PER_LONG;
  				continue;
  			}
  
              // 前面介绍过 fd_set的数组元素是 unsigned long, 即一个元素可表示64个fd, 这里依次遍历这64个bits
  			for (j = 0; j < BITS_PER_LONG; ++j, ++i, bit <<= 1) {
  				struct fd f;
  				if (i >= n)
  					break;
  				if (!(bit & all_bits))
  					continue;
                  // 走到这里，说明当前bit位上有监控的fd
                  
  				f = fdget(i);
  				if (f.file) {
                      // 针对当前fd, 设置其需要监控的事件
  					wait_key_set(wait, in, out, bit,
  						     busy_flag);
                      
                      // vfs_poll 是重中之重，我们下一节会单独讲解，这里先说明它所作的事
                      // 1.  初始化wait entry, 将其加入到这个fd对应的socket的等待队列中
                      // 2.  获取当前socket是否有读，写，异常等事件并返回
  					mask = vfs_poll(f.file, wait);
  
  					fdput(f);
                      // 按位与，看是否有相关事件
  					if ((mask & POLLIN_SET) && (in & bit)) {
  						res_in |= bit;
  						retval++;
  						wait->_qproc = NULL;
  					}
  					if ((mask & POLLOUT_SET) && (out & bit)) {
  						res_out |= bit;
  						retval++;
  						wait->_qproc = NULL;
  					}
  					if ((mask & POLLEX_SET) && (ex & bit)) {
  						res_ex |= bit;
  						retval++;
  						wait->_qproc = NULL;
  					}
                      
  					/* got something, stop busy polling */
  					if (retval) {
  						can_busy_loop = false;
  						busy_flag = 0;
  
  					/*
  					 * only remember a returned
  					 * POLL_BUSY_LOOP if we asked for it
  					 */
  					} else if (busy_flag & mask)
  						can_busy_loop = true;
  
  				}
  			}
              
              // 按unsigned long赋值给返回值数组元素
  			if (res_in)
  				*rinp = res_in;
  			if (res_out)
  				*routp = res_out;
  			if (res_ex)
  				*rexp = res_ex;
              
              // 这里有两层for循环，这里主动出让CPU, 进行一次调度
  			cond_resched();
  		}
  		wait->_qproc = NULL;
      
          // 四种情况下会返回
         // 1.  任意监控的fd上有事件发生
         //  2. 超时
         //  3. 有中断发生
         //  4. wait queue相关操作发生错误
  		if (retval || timed_out || signal_pending(current))
  			break;
  		if (table.error) {
  			retval = table.error;
  			break;
  		}
  
  	  ......
          
          //  当前监控的fd上没有事件发生，也没有超时或中断发生，
         //   将当前进程设置为 TASK_INTERRUPTIBLE， 并调用 schedule
         //   等待事件发生时，对应的socket将当前进程唤醒后，从 这里
        //   继续运行
  		if (!poll_schedule_timeout(&table, TASK_INTERRUPTIBLE,
  					   to, slack))
  			timed_out = 1;
  	}
```

  简单总结如下：

  a. 循环遍历每一个监控的fd；

  b. 有下列情况之一则返回：

```
         1.  任意监控的fd上有事件发生
         2. 超时
         3. 有中断发生
         4. wait queue相关操作发生错误
```

  c. 如查无上述情况，将当前进程设置为 TASK_INTERRUPTIBLE， 并调用 `schedule`作进程切换;

  d. 等待socket 事件发生，对应的socket将当前进程唤醒后，当前进程被再次调度切换回来，继续运行;

  细心的你可能已经发现，这个有个影响效率的问题：即使只有一个监控中的fd有事件发生，当前进程就会被唤醒，然后要将所有监控的fd都遍历一边，依次调用`vfs_poll`来获取其有效事件，好麻烦啊～～～

###### vfs_poll 讲解

* 调用层级

![vfs_poll.png](https://upload-images.jianshu.io/upload_images/2020390-cbd4605514dd2051.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


* 作用：

```
  1.  初始化wait entry, 将其加入到这个fd对应的socket的等待队列中
  2.  获取当前socket是否有读，写，异常等事件并返回
```

* 加入等待队列时，最终会调用 `fs_select.c`中的 `__pollwait`

```
  static void __pollwait(struct file *filp, wait_queue_head_t *wait_address,
  				poll_table *p)
  {
  	struct poll_wqueues *pwq = container_of(p, struct poll_wqueues, pt);
  	struct poll_table_entry *entry = poll_get_entry(pwq);
  	if (!entry)
  		return;
  	entry->filp = get_file(filp);
  	entry->wait_address = wait_address;
  	entry->key = p->_key;
  	init_waitqueue_func_entry(&entry->wait, pollwake);
  	entry->wait.private = pwq;
      
      // 加入到socket的等待队列
  	add_wait_queue(wait_address, &entry->wait);
  }
```

#### 总结

* `select`调用中每类`fd_set`中最多容纳1024个fd；
* 每次调用`select`都需要将三类`fd_set`从用户空间复制到内核空间;
* `wait queue`是个好东西，`select`会被当前进程task加入到每一个监控的socket的等待队列;
* `select`进程被唤醒后即使只有一个被监控的fd有事件发生，也会再次将所有的监控fd遍历一次;
* 在遍历fd的过程中会调用`cond_resched（）`来主动出让CPU, 作进程切换;	