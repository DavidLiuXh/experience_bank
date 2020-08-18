
###  Linux PID 一网打尽

#### 前言

Linux 进程 PID 大家都知道，`top`命令就可以很容易看到各个进程的 PID, 稍进一步`top -H`，我们还能够看到各个线程的ID, 即TID。今天我们想深入到Linux Kernel, 看一看在 Kernel里PID的来龙去脉。

阅读本文 ，您可以了解到：

* 什么是tid, pid, ppid, tgid, pgid, seesion id;
* 内核中是如何表示上面这一系列id的；
* 什么是 pid namesapce;
* 如何创建一个pid namespace以及如何进入一个已存在的pid namespace;
* 内核数据结构task_struct与这一系列id之间的联系;

#### 进程相关的各种ID

##### 进程，线程，线程组，进程组，Session

* 我们在写代码的时候，经常提到进程，线程，但是从Kernel的角度来看，不会区分进程，线程，它们最终都会对应到内核对象`task_struct`上，它也是CPU调度的实体。 不管是进程PID还是线程TID，都是用`task_struct::pid`表示；
* 线程组，顾名思义就是一组有关联的线程，从用户空间角度来说就是一个包含了多线程的进程，细分的话又可以分为主线程和其他线程，其中线程组ID，我们称之为`TGID`，它就等于这个主线程的TID，用`task_struct::tgid`表示；
* 进程组，我们启动一个进程，此进程又创建一个线程，那这个进程和这个线程就属于同一个进程组; 我们启动的这个进程又fork了一个新的进程，那这两个进程和这个线程也是同属于同一个进程组；且这个进程组的ID. 即PGID就是我们最早启动的进程的PID; 

* Session, 简单来说就是一系列进程组的组合。 我们开启一个Shell终端，也就建立了一个Session; 

  我们通过这个shell启动若干个进程，这些进程和这个Shell终端就同属于同一个 Session，这个Session的ID就是这个shell终端进程的PID。

* 我们用一个实例来说明：我们在shell下启动一个名为`thread_test`的进程，这个进程首先创建一个线程，然后再 fork出一个进程，具体代码可以参考[这里](https://github.com/DavidLiuXh/ExampleBank/blob/master/thread_test/thread_test.cc)， 然后我们来查看这个`thread_test`相关的各种id信息：

  

  ```c++
  lw@lw-OptiPlex-7010:~$ ps  -T -eo tid,pid,ppid,tgid,pgid,sid,comm | grep thread_test
    tid     pid     ppid    tgid    pgid    sid      comm
  1294928 1294928 1292449 1294928 1294928 1292449 thread_test (主进程)
  1294929 1294928 1292449 1294928 1294928 1292449 thread_test (创建的线程）
  1294933 1294933 1294928 1294933 1294928 1292449 thread_test (fork的新进程)
  ```

  我们看到：

  1. 主进程，新创建的线程和fork出的新进程，它们的tid均不相同，这是它们各自的真正的唯一标识；

  2. 主进程和创建的线程它们的pid和tgid是一样的，是因为它两个属于同一个线程组，且这个线程组的ID就是主进程的TID;

  3. 主进程和创建的线程的父进程ID，即ppid是一样的（1292449），这个ID其实就是bash shell进程的PID; 

     fork的新进程的父 ID是 上面的主进程ID, 原因很简单是fork出来的嘛；

  4. 主进程，新创建的线程和fork出的新进程，它们的进程组id，即pgid是一样的，都是这个主进程的pid, 因为它们同属于同一个进程组；

  5. 主进程，新创建的线程和fork出的新进程，它们的session id也是一样的，因为它们都是在这个shell终端下产生的，其sid就是终端bash shell进程的PID; 

* 图示说明：

![pid.png](https://upload-images.jianshu.io/upload_images/2020390-38f5c3b061148614.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 内核中进程相关ID的表示

我们以Linux Kernel 5.4.2 为例介绍

##### 你想象中的进程pid的样子

* 我们在写代码时偶尔会需要获取进程的pid和父进程id, 这通常通过`getpid`和`getppid`来获取到，它们返回的`pid_t`类型其实就是个`int`类型；
* 如果我们据此认为内核里的pid也是这么简单的一个POD类型 ，那我们是不是把内核想得过于简单了？好了，我们接着往下看

##### Kernel中的pid

* PID Namespace

  Linux Kernel为了实现资源隔离和虚拟化，引入了Namespace机制，比如docker就充分利用了Namespace机制，相关具体信息感兴趣的同学自行上网搜索吧。

  我们这里主要涉及到PID Namespace, 简单来讲就是可以创建多个pid相互隔离的namespace，其主要特征如下：

  1.  每个namespace里都有自己的1号init进程，来负责启动时初始化和接收孤儿进程；

  2. 不同pid namespace中的各进程pid可以相同；

  3. pid namespace可以层级嵌套创建；下一级pid namespace中进程对其以上所有层的pid namespace都是可见的,同一个task(内核里进程，线程统一都用task_struct表示)且在不同层级namespace中pid可以不同。

  4. 图示说明：

![pid_namespace.png](https://upload-images.jianshu.io/upload_images/2020390-3f75b6fa5274f780.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


  为了表述方便 ，我们将PID Namespace简化为PN
  从上图我们可以看到 :
  PN1-PID2, PN2-PID10和PN4-PID1它们都指向的是同一个task 1;
  PN1-PID3, PN2-PID8和PN5-PID1它们都指向的是同一个task 2;
  
由此我们可以得出一个结论：在Kernel中一个task的ID由两个元素 
 唯一确定 [pid namespace, processid id]，在内核中用`upid`表示：

```c
     struct upid {
        int nr;  // process id
        struct pid_namespace *ns; // 所属的pid namespace
     }
```
     
  5.   如果还是觉得太抽象 ，我们来个实例：
a.  我们先启动一个bash：

```shell
      lw@lw-OptiPlex-7010:~/opensources/ExampleBank/thread_test$ echo $$
      1292449
```

   可以看到这个bash进程ID是 1292449，我们将这一层作为pid namespace 1;

   b. 使用`unshare`创建一个新的pid namespace, 并且启动一个新的bash进程：

```shell
      lw@lw-OptiPlex-7010:~/opensources/ExampleBank/thread_test$ sudo unshare --pid --mount-proc --fork /bin/bash 
      root@lw-OptiPlex-7010:/home/lw/opensources/ExampleBank/thread_test# echo $$
      1
```
可以看到当前bash进程ID是1, 是在一个新的pid namespace中，我们将这一层作为pid namespace 2;
      
c. 我们在pid namespace 2的bash中再创建一个新的pid namespace:
```shell
      lw@lw-OptiPlex-7010:~/opensources/ExampleBank/thread_test$ sudo unshare --pid --mount-proc --fork /bin/bash
      root@lw-OptiPlex-7010:/home/lw/opensources/ExampleBank/thread_test# echo $$
      1
```
可以看到当前bash进程ID是1, 是在一个新的pid namespace中，我们将这一层作为pid namespace 3;

  d.  我们在pid namespace 3的bash中在后台运行vim

```shell
root@lw-OptiPlex-7010:/home/lw/opensources/ExampleBank/thread_test# vim 1 &
      [1] 8
 ```
可以看到这个vim在当前pid namespace 3中的pid是8;

e. 新开启一个bash, 我们查看一下在顶层pid namespace 1中的pid:

```shell
    lw@lw-OptiPlex-7010:~$ pstree -pl | grep bash |grep unshare | grep -v grep 
             |                    |-bash(1292449)---sudo(1565473)---unshare(1565476)---bash(1565477)---sudo(1565555)---unshare(1565556)---bash(1565557)---vim(1565982)
```
可以看到在顶层pid namespace 1中：
  pid namespace 1中启动的bash的pid是1292449
  pid namespace 2中启动的bash的pid是1565477
   pid namespace 3中启动的bash的pid是1565557
      
f. 在顶层pid namespace 1的bash中查看`vim 1`的pid 是 1565982：
```shell
      lw@lw-OptiPlex-7010:~$ ps aux |grep 'vim 1' | grep -v grep 
      root     1565982  0.0  0.1  64980 19980 pts/2    T    18:56   0:00 vim 1
```
g. 在pid namespace 2的bash中查看`vim 1`的pid 是 17:
```shell
      //使用nsenter先进入pid namespace2, 这个 1565477 即为pid namespace 2中bash进程的PID 	
      lw@lw-OptiPlex-7010:~$ sudo nsenter --pid --mount -t 1565477
      root@lw-OptiPlex-7010:/# ps aux |grep 'vim 1' | grep -v grep 
      root          17  0.0  0.1  64980 19980 pts/2    T    18:56   0:00 vim 1
```
h. 在pid namespace 3的bash中查看`vim 1`的pid 是 8:
```
      //使用nsenter先进入pid namespace2
      lw@lw-OptiPlex-7010:~$ sudo nsenter --pid --mount -t 1565557
      root@lw-OptiPlex-7010:/# ps aux |grep 'vim 1' | grep -v grep 
      root          8  0.0  0.1  64980 19980 pts/2    T    18:56   0:00 vim 1
```
 
* PID大一统

  前面我们说过了，进程相关的ID除了PID(TID)，还有TDID, PGID, SID(Session ID), 在kernel中它们都被大一统起来，用`struct pid`表示, 它定义在`incluse/linux/pid.h`中；

```c
  enum pid_type         
  {                     
  	PIDTYPE_PID,  
      PIDTYPE_TGID, 
  	PIDTYPE_PGID, 
  	PIDTYPE_SID,  
  	PIDTYPE_MAX,  
  };
  struct pid                                         
  {                                                  
  	refcount_t count;                          
  	unsigned int level;                        
  	/* lists of tasks that use this pid */     
  	struct hlist_head tasks[PIDTYPE_MAX];      
  	/* wait queue for pidfd notifications */   
  	wait_queue_head_t wait_pidfd;              
  	struct rcu_head rcu;                       
  	struct upid numbers[1];                    
  };
```

  1. count : 引用计数，因为这个pid变量可以代码这个task的pid(tid), 同时还可以是一组task的TGID, 甚至是一组进程的PGID, 所以其每被引用一次，引用计数加1；引用计数降为0时，代表其已无task引用，可以free掉了；
  2. level : 上面的namepsace图中我们已经看到pid namespace是一个层级结构，这个level就表示当前这个pid属于第几层，其上面还有level - 1层；
  3. struct upid numbers[1] ：初看起来是一个只有一个upid元素的数组，其实它是属于struct尾部的类似于0元素数组的动态数组，每次在分配`struct pid`时，`numbers`会被扩展到`level`个元素；
它用来容纳在每一层pid namespace中的 id; 
  4. struct hlist_head tasks[PIDTYPE_MAX] : 前面我们也已经说过，进程相关的ID除了PID(TID)，还有TDID, PGID, SID(Session ID)，因此这个`tasks`是一个hash链表的数组，

#### 内核PID与PID Namespace

经过前面那么多的铺垫，我们知道了如下两件事：

* PID Namespace是可以分级嵌套的；
* 同一个task在各级子PID Namespace和父PID Namespace中的进程ID是不同的。

在内核中进程的PID采用`struct pid`结构体来表示，这就需要在`struct pid`结构体中有成员可以保存这个task在各级PID Namespace中的对应的不同的进程ID:

```c
struct pid                                         
{                     
    ...
	unsigned int level; //表示当前进程里在第几级pid namespace中创建
    ...
	struct upid numbers[1]; //用于保存在每级pid namespace中的进程ID（struct upid）     
};
```

##### 内核PID结构体的创建

* 看了`struct pid`结构体的声明，您可能会觉得奇怪，明明说了`numbers`数组保存每级pid namespace中的进程ID，但是为什么这个数据只有一个元素？

  这个是Kernel里惯用的手法，结构体最后一个元素可以声明为动态数组，动态数组一般是在创建这个结构体时被动态分配内存。我们来看一下`struct pid`结构体的创建和初始化，这要追溯到task的创建，最终会追溯到定义在`kernel/fork.c`中的`copy_process`函数，它的具体流程我们在这里不详述，只看相关的pid部分：

```c
  static __latent_entropy struct task_struct *copy_process(              
  			struct pid *pid,
  			int trace,
  			int node,
  			struct kernel_clone_args *args) {
                  if (pid != &init_struct_pid) {                             
                      pid = alloc_pid(p->nsproxy->pid_ns_for_children);  
                      if (IS_ERR(pid)) {                                 
                          retval = PTR_ERR(pid);                     
                          goto bad_fork_cleanup_thread;              
                      }                                                  
                  }         
  			}
```

  下面我们来过一个`alloc_pid`函数，它定义在 `kernel/pid.c`中：

```c
  struct pid *alloc_pid(struct pid_namespace *ns)
  {
      //分配内存并构造struct pid
    	pid = kmem_cache_alloc(ns->pid_cachep, GFP_KERNEL);
      if (!pid)
          return ERR_PTR(retval);                    
  
      tmp = ns;                                          
      pid->level = ns->level;
  }
```

  1. 首先使用`kmem_cache_alloc`分配内存并构造struct pid，它使用父pid namespace的pid_cachep来分配内存，这个`pid_cachep`是`kmem_cache`类型 ，它在`create_pid_namepsace`中被创建：

```c
     static struct pid_namespace *create_pid_namespace(struct user_namespace *user_ns,
     ▸       struct pid_namespace *parent_pid_ns)                                     
     {                                                                                
     ▸       struct pid_namespace *ns;             
             //这里确定了当前pid namespace的level, 是父pid namespace level + 1
     ▸       unsigned int level = parent_pid_ns->level + 1;      
             ...
             //分配当前pid namespace结构体所需的内存
             ns = kmem_cache_zalloc(pid_ns_cachep, GFP_KERNEL); 
             ...
             //创建上面所说的 kmem_cache, 将当前level作为参数传入
             ns->pid_cachep = create_pid_cachep(level); 
             ...
     }
```
     
  2. 接着再来看一下`create_pid_cachep`:
  
```c
     static struct kmem_cache *create_pid_cachep(unsigned int level)                
     {                                                                              
     ▸       /* Level 0 is init_pid_ns.pid_cachep */                                
     ▸       struct kmem_cache **pkc = &pid_cache[level - 1];                       
     ▸       struct kmem_cache *kc;                                                 
     ▸       char name[4 + 10 + 1];                                                 
     ▸       unsigned int len;                                                      
                                                                                    
     ▸       kc = READ_ONCE(*pkc);                                                  
     ▸       if (kc)                                                                
     ▸       ▸       return kc;                                                     
                                                                                    
     ▸       snprintf(name, sizeof(name), "pid_%u", level + 1);   
             //这里是关键，计算需要分配的内存，除了 sizeof(struct pid)
             //还需要加上 level * sizeof(struct upid)， 这个就是上面提到的
             //动态数据的大小，按level的大小来分配
     ▸       len = sizeof(struct pid) + level * sizeof(struct upid);                
     ▸       mutex_lock(&pid_caches_mutex);                                         
     ▸       /* Name collision forces to do allocation under mutex. */              
     ▸       if (!*pkc)                                                             
     ▸       ▸       *pkc = kmem_cache_create(name, len, 0, SLAB_HWCACHE_ALIGN, 0); 
     ▸       mutex_unlock(&pid_caches_mutex);                                       
     ▸       /* current can fail, but someone else can succeed. */                  
     ▸       return READ_ONCE(*pkc);                                                
     }                                                                              
```
  
     到此，我们解决了`struct pid`中`numbers`的分配问题。

* 图示说明一下`struct pid`和PID Namespace的关系

![pid_namespace_task.png](https://upload-images.jianshu.io/upload_images/2020390-c06b15b77be7b9e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


​       如上图所示：

​            一个task_strcut表示一个 进程或线程，其`thread_pid`成员变量指向它的`struct pid`结构体；
​            `  struct pid`结构体的`numbers`数组有三个元素，分别表示在PID Namespace 1,2,3中的三个uid;

#### 内核PID与TID,TGID,PGID的关系

##### task_struct结构体中各种相关ID的存储

* 我们知道在用户空间是进程和线程之分，创建了一个进程，里面具体作事的是这个进程包含的第一个线程，也叫主线程；在主线程里又可以创建新的线程，线程里又可以继续创建线程，当前的进程还可以fork出新的进程；

* 但是在内核中，只有`task`。进程，线程，主线程统统用`task_struct`结构体表示；

* 如果创建的是进程，这个进程PID是由它创建的所有线程的线程组ID,即TGID

  `taks_struct`中有个成员变量`struct signal_struct *signal`，这个`struct signal_struct`也有个成员变量`struct pid *pids[PIDTYPE_MAX];`，如果当前的task表示进程，那这个`pids数组`用来存储PID, TGID, PGID, SID，其赋值是在`copy_process`中进行：

```c
  //当前创建的p是进程
  if (thread_group_leader(p)) {    
          //新生成的pid就是这个线程组的TGID
  ▸       init_task_pid(p, PIDTYPE_TGID, pid);    
          //新生成的task和current task是同属于一个进程组，有相同的PGID
  ▸       init_task_pid(p, PIDTYPE_PGID, task_pgrp(current));   
          //新生成的task和current task是同属于一个Session，有相同的session id
  ▸       init_task_pid(p, PIDTYPE_SID, task_session(current));    
                                                                   
  ▸       ...  
  ▸       /*                                                       
  ▸        * Inherit has_child_subreaper flag under the same       
  ▸        * tasklist_lock with adding child to the process tree   
  ▸        * for propagate_has_child_subreaper optimization.       
  ▸        */                                                      
  ▸       ...   
  ▸       attach_pid(p, PIDTYPE_TGID);                             
  ▸       attach_pid(p, PIDTYPE_PGID);                             
  ▸       attach_pid(p, PIDTYPE_SID);                              
  ▸       __this_cpu_inc(process_counts);                          
  }
```

  我们来重点看下`attach_pid`的实现：

```c
  void attach_pid(struct task_struct *task, enum pid_type type)          
  {                                                                      
  ▸       struct pid *pid = *task_pid_ptr(task, type);                   
  ▸       hlist_add_head_rcu(&task->pid_links[type], &pid->tasks[type]); 
  }                                                                      
```

  主要作用就是能过调用`hlist_add_head_rcu`把当前task连接入pid->tasks对应的hash链表;

  我们以`PIDTYPE_PGID`来举例说明，同属于一个进程组的所有进程对应的`taks_struct`都被链接到同一个hash链表上：

![pgid.png](https://upload-images.jianshu.io/upload_images/2020390-ec4fc33ca3e696fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##### 线程组的管理

从上面的介绍我们知道PID还有一种类型是`PIDTYEP_TGID`, 对应的是线程组，但是从代码实现来看同属一个线程组的所有线程并没有使用上一节中的`pid::taks[PIDTYE_TGID]`和`task_struct::pid_links[PIDTYPE_TGID]`来链接起来。

在`copy_process`中有如下一段代码：

``` C
if (clone_flags & CLONE_THREAD) {                 
▸       p->exit_signal = -1;                      
▸       p->group_leader = current->group_leader;  
▸       p->tgid = current->tgid;                  
}
```

其中`group_leader`表示的是一个线程组的leader, 也就是一个进程的主线程。

在`copy_process`中还有如下一段代码：

```c
list_add_tail_rcu(&p->thread_group,                    
▸       ▸         &p->group_leader->thread_group);               
```

这段代码就是将当前的线程对应的`task_strcut`链接到其所在进程的主线程的`thread_group`链表中，至此同属于同一个线程组的所有线程对应的`task_struct`就全部链表在同一个双向链表中了：

![tgid.png](https://upload-images.jianshu.io/upload_images/2020390-effcf9709e93acb4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 后记

关于PID的相关内容我们暂时就先讲到这里，希望可以帮大家理清一些基本概念，疏漏之处，望大家指证。