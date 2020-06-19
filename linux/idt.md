[toc]

#### Linux中断一网打尽 —— 中断及其初始化

##### 前情提要

通过本文您可以了解到如下内容：

* Linux 中断是什么，如何分类，能干什么？
* Linux 中断在计算机启动各阶段是如何初始化的？

##### 中断是什么

既然叫**中断**, 那我们首先就会想到这个中断是中断谁？想一想计算机最核心的部分是什么？没错， CPU， 计算机上绝大部分的计算都在CPU中完成，因此这个中断也就是中断CPU当前的运行，让CPU转而先处理这个引起中断的事件，通常来说这个中断的事件比较紧急，处理完毕后再继续执行之前被中断的task。比如，我们敲击键盘，CPU就必须立即响应这个操作，不然我们打字就全变成了慢动作～。说白了中断其实就是一种主动通知机制，如果中断源不主动通知，那想知道其发生了什么事情，只能一次次地轮询了，白白耗费CPU。

##### 中断的分类

大的方向上一般分为两大类：同步中断和异步中断，按Intel的说法，将异步中断称为中断，将同步中断称为异常。

###### 异步中断

主要是指由CPU以外的硬件产生的中断，比如鼠标，键盘等。它的特点是相对CPU来说随时随机发生，事先完全没有预兆，不可预期的。异步中断发生时，CPU基本上都正在执行某条指令。

异步中断可分为可屏蔽和不可屏蔽两种，字如其义不用多解释。

###### 同步中断

主要是指由CPU在执行命令过程中产生的异常，它一定是在CPU执行完一条命令后才会发出，产生于CPU内部。按其被CPU处理后返回位置的不同，我们将同步中断分为故障(fault), 陷阱(trap)和终止(abort)三类。我们通过一个表格来作下对比区分：

| 异步中断分类 | 特点                                 | 处理完毕后的返回位置               | 例子     |
| ------------ | ------------------------------------ | ---------------------------------- | -------- |
| 故障(fault)  | 潜在可能恢复的错误                   | 重新执行引起此故障的指令           | 缺页中断 |
| 陷阱(trap)   | 为了实现某种功能有意而为之发生的错误 | 执行引发当前陷阱的指令的下一条指令 | 系统调用 |
| 终止(abort)  | 不可恢复的错误                       | 没有返回，进程将被终止             |          |

两点说明：

* `处理完毕后的返回位置`：发生异常时，CPU最终会进入到相应的异常处理程序中（简单说就是CPU需要执行一次跳转）在执行具体操作前会设置好的异常处理完成后跳转回的CS:IP, 即代码段寄存器和程序指针寄存器，不同类型的异常其设置的CS:IP不同而已；

* 有些分类方法还会有一种叫`可编程异常`的，比如说把系统调用算作这一类，也可以。但是如果按处理完毕后的返回位置来说系统调用是可以归入`陷阱`这一类的。

##### 硬件中断的管理模型

我们都知道CPU上只有有限多的脚针，负责与外部通讯，比如有数据线，地址线等，也有中断线，但一般只有两条NMI(不可屏蔽中断线)和INTR(可屏蔽中断线)， 新的CPU有LINT0和LINT1脚针。那您会问了，电脑上有那么多外设，CPU就这两根线，怎么接收这么多外设的中断信号呢？确实，因此CPU找了一个管理这些众多中断的代理人——中断控制器。

就目前我们使用的SMP多核架构里，我们经常使用高级可编程中断控制器APIC， 老式的 8259A 可编程中断控制器大家有兴趣可自行搜索。

APIC分为两部分，IO APIC和Local APIC，从名字上我们就可略知一二。

* IO APIC: 用来连接各种外设的硬件控制器，接收其发送的中断请求信号，然后将其传送到Local APIC, 这个IO APIC一般会封装在主板南板芯片上; 

* Local APIC: 基本上集成在了CPU里， 向CPU通知中断发生。

* 放张网上的图：

  ![ioapic.jpg](https://upload-images.jianshu.io/upload_images/2020390-a7d1901cec573d06.jpg?imageMogr2/auto-orient/strip)


##### 中断的初始化

###### Linux 启动流程

中断的初始化是穿插在Linux本身启动和初始化过程中的，因此我们在这里简要说一下Linux本身的初始化。

* 64位Linux启动大的方向上需要经过 `实模式 -> 保护模式 -> 长模式` 第三种模式的转换;
* 电源接通，CPU启动并重置各寄存器后运行于实模式下，CS:IP加载存储于ROM中的一跳转指令，跳转到BIOS中；
* BIOS启动，硬件自测，读取MRB;
* BIOS运行第一阶段引导程序，第一阶段引导程序运行第二阶段引导程序，通常是 grub; 
* Grub开始引导内核运行;
* 相关初始化后进行保护模式，再进入长模式，内核解压缩；
* 体系无关初始化部分;
* 体系相关初始化部分;

总结了一张图，仅供参考：

![linux启动流程.png](https://upload-images.jianshu.io/upload_images/2020390-a06ac93a2c424d15.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


###### 中断描述符表

外设千万种，CPU统统不知道。所有的中断到了CPU这里就只是一个中断号，然后初始化阶段设置好中断号到中断处理程序的对应关系，CPU获取到一个中断号后，查到对应的中断处理程序调用就好了。

这两者的对应关系最后会抽象成了`中断向量表`， 现在叫 `IDT`中断描述符表。

###### 中断的第一次初始化

`实模式下`的初始化

* 上面那张Linux启动流程图如果你仔细看的话会发现在BIOS程序加载运行时，在实模式下也有一个BIOS的中断向量表，这个中断向量表提供了一些类似于BIOS的系统调用一样的方法。比如Linux在初始化时需要获取物理内存的详情，就 是调用了BIOS的相应中断来获取的。见下图：

![选区_035.png](https://upload-images.jianshu.io/upload_images/2020390-c8508006e425ab21.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 中断的第二次初始化

* 在进入到`保护模式`后，会全新初始化一个空的中断描述符表 IDT, 供 kernel 使用;

* Linux Kernel提供256个大小的中断描述符表

  ```assembly
  #define IDT_ENTRIES			256
  
  gate_desc idt_table[IDT_ENTRIES] __page_aligned_bss;
  ```

###### 中断的第三次初始化

* 在进入到`长模式`后，在`x86_64_start_kernel`先初始化前32个异常类型的中断(即上面定义的 idt_table 的前32项)；

```c
  void __init idt_setup_early_handler(void)
  {
  	int i;
  
  	for (i = 0; i < NUM_EXCEPTION_VECTORS; i++)
  		set_intr_gate(i, early_idt_handler_array[i]);
  
  	load_idt(&idt_descr);
  }
```

  其中 `early_idt_handler_array`这个数组放置了32个异常类型的中断处理程序，我们先看一下它的定义：

```c
  const char early_idt_handler_array[32][9];
```

  二维数组，每一个`early_idt_handler_array[i]`有9个字节。

  这个 `early_idt_handler_array`的初始化很有意思，它用AT&T的汇编代码完成，在文件`arch/x86/kernel/head_64.S`中：

```assembly
  ENTRY(early_idt_handler_array)
  	i = 0
  	.rept NUM_EXCEPTION_VECTORS
  	.if ((EXCEPTION_ERRCODE_MASK >> i) & 1) == 0
  		UNWIND_HINT_IRET_REGS
  		pushq $0	# Dummy error code, to make stack frame uniform
  	.else
  		UNWIND_HINT_IRET_REGS offset=8
  	.endif
  	pushq $i		# 72(%rsp) Vector number
  	jmp early_idt_handler_common
  	UNWIND_HINT_IRET_REGS
  	i = i + 1
  	.fill early_idt_handler_array + i*EARLY_IDT_HANDLER_SIZE - ., 1, 0xcc
  	.endr
  	UNWIND_HINT_IRET_REGS offset=16
  END(early_idt_handler_array)
 
 ```

  这段汇编循环遍历32次来初始化每一个`early_idt_handler_array[i]`, 也就是填充它的9个字节：其中2个字节是压栈错误码指令，2个字节是压栈向量号指令，余下的5个字节是函数跳转指令（`jmp early_idt_handler_common`）。由此我们可以看出，这前32个异常类型的中断处理函数最终都会调用到`early_idt_handler_common`, 这个函数这里就不贴它的代码了，我们说下它的大致流程：
  
 ```c
  a. 先将各寄存器的值压栈保存；
  b. 如果是 缺页异常，就调用 `early_make_patable`; 
  c. 如果是 其他异常，就调用 `early_fixup_exception`; 
  ```
  
* 体系结构相关的中断初始化

  这也是一次部分初始化，它发生在 `start_kernel`的`setup_arch`中，即发生在  Linux 启动流程中的体系结构初始化部分。这部分实际上是更新上面已初始化的32个异常类中的X86_TRAP_DB(1号, 用于debug)和X86_TRAP_BP(3号, 用于debug时的断点);

```c
  static const __initconst struct idt_data early_idts[] = {
  	INTG(X86_TRAP_DB,		debug),
  	SYSG(X86_TRAP_BP,		int3),
  };
  
  void __init idt_setup_early_traps(void)
  {
  	idt_setup_from_table(idt_table, early_idts, ARRAY_SIZE(early_idts),
  			     true);
  	load_idt(&idt_descr);
  }
  
```

  `debug`和`int3`这两个汇编实现的中断处理程序这里我们就不详述了。

* 更新 `X86_TRAP_PF 缺页异常`的中断处理程序

```c
  void __init idt_setup_early_pf(void)
  {
  	idt_setup_from_table(idt_table, early_pf_idts,
  			     ARRAY_SIZE(early_pf_idts), true);
  }
  
  static const __initconst struct idt_data early_pf_idts[] = {
  	INTG(X86_TRAP_PF,		page_fault),
  };
```

* 在`trap_init`中调用 `idt_setup_traps`更新部分异常的中断处理程序：

```c
  void __init idt_setup_traps(void)
  {
  	idt_setup_from_table(idt_table, def_idts, ARRAY_SIZE(def_idts), true);
  }
  
  static const __initconst struct idt_data def_idts[] = {
  	INTG(X86_TRAP_DE,		divide_error),
  	INTG(X86_TRAP_NMI,		nmi),
  	INTG(X86_TRAP_BR,		bounds),
  	INTG(X86_TRAP_UD,		invalid_op),
  	INTG(X86_TRAP_NM,		device_not_available),
  	INTG(X86_TRAP_OLD_MF,		coprocessor_segment_overrun),
  	INTG(X86_TRAP_TS,		invalid_TSS),
  	INTG(X86_TRAP_NP,		segment_not_present),
  	INTG(X86_TRAP_SS,		stack_segment),
  	INTG(X86_TRAP_GP,		general_protection),
  	INTG(X86_TRAP_SPURIOUS,		spurious_interrupt_bug),
  	INTG(X86_TRAP_MF,		coprocessor_error),
  	INTG(X86_TRAP_AC,		alignment_check),
  	INTG(X86_TRAP_XF,		simd_coprocessor_error),
  
  #ifdef CONFIG_X86_32
  	TSKG(X86_TRAP_DF,		GDT_ENTRY_DOUBLEFAULT_TSS),
  #else
  	INTG(X86_TRAP_DF,		double_fault),
  #endif
  	INTG(X86_TRAP_DB,		debug),
  
  #ifdef CONFIG_X86_MCE
  	INTG(X86_TRAP_MC,		&machine_check),
  #endif
  
  	SYSG(X86_TRAP_OF,		overflow),
  #if defined(CONFIG_IA32_EMULATION)
  	SYSG(IA32_SYSCALL_VECTOR,	entry_INT80_compat),
  #elif defined(CONFIG_X86_32)
  	SYSG(IA32_SYSCALL_VECTOR,	entry_INT80_32),
  #endif
  };
```

* 在`trap_init`中调用 `idt_setup_ist_traps`更新部分异常的中断处理程序,

  看到这里您可能问，上面不是调用了`idt_setup_traps`，怎么这时又调用`idt_setup_ist_traps`? 这两者有什么区别？说起来话有点长，我们尽量从流程上给大家讲清楚，但不深入到具体的细节。

  1. 想说明这个问题，我们先来讲下栈这个东西:

     a. 首先每个进程都有自己的用户态栈，对应进程虚拟地址空间内的stack部分，用于进程在用户态变量申请，函数调用等操作；

     b. 除了用户态栈，每个进程在创建时(内核对应创建 task_struct结构)同时会创建对应的内核栈，这里进程由用户态进入到内核态执行函数时，相应的所用的栈也会切换到内核栈；

     c. 如果内核进入到中断处理程序，早期的kernel针对中断处理程序的执行会使用当前中断task的内核栈，这里有存在一定的问题，存在栈溢出的风险。举个例子，如果在中断处理程序里又发生了异常中断，此时会触发`double fault`，但其在处理过程中依然要使用当前task的内核栈，并且当前task内核栈已满，`double fault`无法被正确处理。为了解决这样的内部，linux kernel引出了独立的内核栈，针对SMP系统，它还是pre-cpu的。我们来看一下其初始化：

 ```c
     void irq_ctx_init(int cpu)
     {
     	union irq_ctx *irqctx;
     
     	if (hardirq_ctx[cpu])
     		return;
     
         // 硬中断独立栈
     	irqctx = (union irq_ctx *)&hardirq_stack[cpu * THREAD_SIZE];
     	irqctx->tinfo.task		= NULL;
     	irqctx->tinfo.cpu		= cpu;
     	irqctx->tinfo.preempt_count	= HARDIRQ_OFFSET;
     	irqctx->tinfo.addr_limit	= MAKE_MM_SEG(0);
     
     	hardirq_ctx[cpu] = irqctx;
     
         //软中断独立栈
     	irqctx = (union irq_ctx *)&softirq_stack[cpu * THREAD_SIZE];
     	irqctx->tinfo.task		= NULL;
     	irqctx->tinfo.cpu		= cpu;
     	irqctx->tinfo.preempt_count	= 0;
     	irqctx->tinfo.addr_limit	= MAKE_MM_SEG(0);
     
     	softirq_ctx[cpu] = irqctx;
     
     	printk("CPU %u irqstacks, hard=%p soft=%p\n",
     		cpu, hardirq_ctx[cpu], softirq_ctx[cpu]);
     }
 ```

     可以看到还特别贴心地为softirq也开辟了单独的栈。

2. 在x86_64位系统中，还引入了一种新的栈配置：IST(Interrupt Stack Table)。目前Linux kernel中每个cpu最多支持7个IST，可以通过tss.ist[]来访问。

3. 现在我们再来看`idt_setup_ist_traps`，其实就是重新初始化一个异常处理，让这些异常处理使用IST作为中断栈。

```assembly
   void __init idt_setup_ist_traps(void)
   {
   	idt_setup_from_table(idt_table, ist_idts, ARRAY_SIZE(ist_idts), true);
   }
   
   static const __initconst struct idt_data ist_idts[] = {
   	ISTG(X86_TRAP_DB,	debug,		IST_INDEX_DB),
   	ISTG(X86_TRAP_NMI,	nmi,		IST_INDEX_NMI),
   	ISTG(X86_TRAP_DF,	double_fault,	IST_INDEX_DF),
   #ifdef CONFIG_X86_MCE
   	ISTG(X86_TRAP_MC,	&machine_check,	IST_INDEX_MCE),
   #endif
   };
   
   #define ISTG(_vector, _addr, _ist)			\
   	G(_vector, _addr, _ist + 1, GATE_INTERRUPT, DPL0, __KERNEL_CS)
```

   其中 `IST_INDEX_DB` `IST_INDEX_NMI` `IST_INDEX_DF` `IST_INDEX_MCE`就是要使用的ist[]的索引。

* 剩下的最后一部分就是硬件中断的初始化了，它同样在`start_kernel`中执行：

```c
  early_irq_init();
  init_IRQ();
```

  这部分具体细节我们在[Linux中断一网打尽(2) - IDT及中断处理的实现](https://www.jianshu.com/p/a799eb285328)介始。

