---
title: Linux-0.11内核main函数执行过程分析——1
tags: 
- linux
- 底层
categories: 
- Linux 内核
date: 2019-10-12 09:06:11
---


## 概述

本文在结合赵炯博士的《Linux内核0.11完全注释》和对应源代码的基础上，总结而成。

Linux0.11的大致启动流程如下：
1. pc机器的BIOS程序启动，开始加载硬盘中的bootsect.s对应的机器码程序到内存中，将CPU的程序计数器指向bootsect.s对应机器码
在内存中的起始位置，开始执行bootsect.s代码。
2. 执行bootsect.s代码的过程中，从硬盘加载了setup.s对应的机器码到程序的内存中，并跳转到setup.s机器码的开始处开始执行setup.s
对应机器码。setup的主要工作是将硬盘，内存等硬件的参数信息读取并放置到约定的安全位置。加载head.s和main.c共同组成的system模块
机器码到内存中，将程序计数器指定到system代码开头的地方，指定CPU进入保护模式工作。准备开始执行head.s代码。
3. head.s对应的机器码主要完成全局描述符表、中断描述符表、页表等信息的内存分配工作，具体的赋值需要到后续完成。
4. 执行完head.s之后，就该执行main.c函数中的内容了，这也是本文要分析的代码内容。
```
after_page_tables:
	push 0			;// These are the parameters to main :-)
	push 0			;// 这些是调用main 程序的参数（指init/main.c）。
	push 0
	push L6			;// return address for main, if it decides to.
	push _main_rename		;// '_main'是编译程序对main 的内部表示方法。
	jmp setup_paging
```

之所以有这一段流程分析，是为了有个上下文，因为对于好奇的同学来讲，总要问机器是怎么执行到main函数的，之前都干了什么。有了这里的上下文，
感兴趣的同学就可以自行去探索一下上述几个汇编程序的内容。

还有一点需要说明，对于汇编语言，本身大家都学习过，在遇到不懂的助记符的时候，百度一下，很多时候都能找到答案。我的博客中也有我在看汇编代码时候
在网上查找到的一些汇编语言的知识记录，本身赵炯博士的书里也较为系统的介绍了一部分重要的语言特性，大家感兴趣可以看看。

## main函数的主体内容

``` c++

本节是Linux0.11代码中main函数的内容，方便大家阅读添加到这里。后续的内容都是以下面这段代码为脉络的。在这里，先分析一下
在调用mem_init函数之前做的一些工作。main函数开始的时候，主要的工作是计算高速缓冲区末端，由此将整个内存分为三个较大的部分，
分别是内核代码区，高速缓冲区和主内存区。其中内核代码区就是内核代码放置的位置。高速缓冲区是用来进行硬盘数据缓冲的，这个区由
缓冲区头和缓冲区块共同组成。缓冲区头是缓冲区块的元数据，用来标记对应缓冲区块是哪个硬盘的哪个扇区的数据缓冲，缓冲区块则存储了
磁盘中多个扇区的完整数据，具体多少个扇区需要根据操作系统的设置来决定，在Linux0.11来说，一个缓冲区块存储了1024字节的数据，而
一个扇区一边是512字节的数据，因此一个缓冲区块存储了两个扇区的数据。第三部分是主内存区，就是用户程序执行过程中分配内存的地方。

void main_rename(void)		/* 这里确实是void，并没错。 */
{			/* 在startup 程序(head.s)中就是这样假设的。 */
/*
 * 此时中断仍被禁止着，做完必要的设置后就将其开启。
 */
	// 下面这段代码用于保存：
	// 根设备号 -> ROOT_DEV； 高速缓存末端地址 -> buffer_memory_end；
	// 机器内存数 -> memory_end；主内存开始地址 -> main_memory_start；
 	ROOT_DEV = ORIG_ROOT_DEV;
 	drive_info = DRIVE_INFO;
	memory_end = (1<<20) + (EXT_MEM_K<<10);// 内存大小=1Mb 字节+扩展内存(k)*1024 字节。
	memory_end &= 0xfffff000;			// 忽略不到4Kb（1 页）的内存数。
	if (memory_end > 16*1024*1024)		// 如果内存超过16Mb，则按16Mb 计。
		memory_end = 16*1024*1024;
	if (memory_end > 12*1024*1024)		// 如果内存>12Mb，则设置缓冲区末端=4Mb
		buffer_memory_end = 4*1024*1024;
	else if (memory_end > 6*1024*1024)	// 否则如果内存>6Mb，则设置缓冲区末端=2Mb
		buffer_memory_end = 2*1024*1024;
	else
		buffer_memory_end = 1*1024*1024;// 否则则设置缓冲区末端=1Mb
	main_memory_start = buffer_memory_end;// 主内存起始位置=缓冲区末端；
#ifdef RAMDISK	// 如果定义了虚拟盘，则主内存将减少。
	main_memory_start += rd_init(main_memory_start, RAMDISK*1024);
#endif
// 以下是内核进行所有方面的初始化工作。阅读时最好跟着调用的程序深入进去看，实在看
// 不下去了，就先放一放，看下一个初始化调用-- 这是经验之谈:)
	mem_init(main_memory_start,memory_end);
	trap_init();	// 陷阱门（硬件中断向量）初始化。（kernel/traps.c）
	blk_dev_init();	// 块设备初始化。（kernel/blk_dev/ll_rw_blk.c）
	chr_dev_init();	// 字符设备初始化。（kernel/chr_dev/tty_io.c）空，为以后扩展做准备。
	tty_init();		// tty 初始化。（kernel/chr_dev/tty_io.c）
	time_init();	// 设置开机启动时间 -> startup_time。
	sched_init();	// 调度程序初始化(加载了任务0 的tr, ldtr) （kernel/sched.c）
	buffer_init(buffer_memory_end);// 缓冲管理初始化，建内存链表等。（fs/buffer.c）
	hd_init();		// 硬盘初始化。（kernel/blk_dev/hd.c）
	floppy_init();	// 软驱初始化。（kernel/blk_dev/floppy.c）
	sti();			// 所有初始化工作都做完了，开启中断。

// 下面过程通过在堆栈中设置的参数，利用中断返回指令切换到任务0。
	move_to_user_mode();	// 移到用户模式。（include/asm/system.h）
	if (!fork()) {		/* we count on this going ok */
		init();
	}
/*
 * 注意!! 对于任何其它的任务，'pause()'将意味着我们必须等待收到一个信号才会返
 * 回就绪运行态，但任务0（task0）是唯一的意外情况（参见'schedule()'），因为任
 * 务0 在任何空闲时间里都会被激活（当没有其它任务在运行时），
 * 因此对于任务0'pause()'仅意味着我们返回来查看是否有其它任务可以运行，如果没
 * 有的话我们就回到这里，一直循环执行'pause()'。
 */
	for(;;) pause();
} // end main
```

### mem_init函数

该函数是对主内存区进行管理初始化的。有以下一些点需要get到：

1. Linux0.11将主内存按照4K的大小划分为多个页面来管理
2. 那么，内核如何知道哪些页面时可用的，哪些页面还为使用？
3. 如何处理内存页面的共享问题？

下面的初始化代码可以大致解答这些问题：
1. 使用全局数组mem_map与主内存中的每个页面进行对应。
2. 使用数字表示页面被使用的次数，当某个下标对应的数字为0的时候，表示
其下标对应页面未使用，可以被分配。
那么，mem_init函数的主要工作就是讲mem_map初始化为0。

``` c++
void mem_init(long start_mem, long end_mem)
{
	int i;

	HIGH_MEMORY = end_mem;// 设置内存最高端。
	for (i=0 ; i<PAGING_PAGES ; i++)// 首先置所有页面为已占用(USED=100)状态，
		mem_map[i] = USED;// 即将页面映射数组全置成USED。
	i = MAP_NR(start_mem);// 然后计算可使用起始内存的页面号。
	end_mem -= start_mem;// 再计算可分页处理的内存块大小。
	end_mem >>= 12;// 从而计算出可用于分页处理的页面数。
	while (end_mem-->0)// 最后将这些可用页面对应的页面映射数组清零。
		mem_map[i++]=0;
}
```
### trap_init

要理解这个函数在做什么，首先要理解中断之于操作系统的意义：

1. CPU在一刻不停的运转，假设我们正在执行用户程序，那么内核怎么来调度CPU来执行其他的程序呢？

2. 在单核CPU中，用户程序正在执行，答案是无法运行内核程序要调度CPU去执行其他的程序。那么内核调度是如何实现的？

3. 答案是，时钟中断。CPU会按照一定的频率去接受来自外部的时钟中断，CPU都会根据时钟中断的中断向量号到中断向量表中查找时钟中断
对应的中断处理程序，时钟中断的中断处理程序会调用内核的调度方法schedule方法，决定是否进行进程切换工作。

4. 讲到这里，我们大概就理解了trap_init函数的作用，就是向中断向量表注册每个中断向量号对应的中断处理函数，以便CPU在接收到中断后能够正确处理中断。

看下面代码，可以看到该函数初始化一些我们可以猜到的一些中断陷入处理老程序0除，调试，溢出，页错误等等。

``` c++
void trap_init(void)
{
	int i;

	set_trap_gate(0,&divide_error);// 设置除操作出错的中断向量值。以下雷同。
	set_trap_gate(1,&debug);
	set_trap_gate(2,&nmi);
	set_system_gate(3,&int3);	/* int3-5 can be called from all */
	set_system_gate(4,&overflow);
	set_system_gate(5,&bounds);
	set_trap_gate(6,&invalid_op);
	set_trap_gate(7,&device_not_available);
	set_trap_gate(8,&double_fault);
	set_trap_gate(9,&coprocessor_segment_overrun);
	set_trap_gate(10,&invalid_TSS);
	set_trap_gate(11,&segment_not_present);
	set_trap_gate(12,&stack_segment);
	set_trap_gate(13,&general_protection);
	set_trap_gate(14,&page_fault);
	set_trap_gate(15,&reserved);
	set_trap_gate(16,&coprocessor_error);
// 下面将int17-48 的陷阱门先均设置为reserved，以后每个硬件初始化时会重新设置自己的陷阱门。
	for (i=17;i<48;i++)
		set_trap_gate(i,&reserved);
	set_trap_gate(45,&irq13);// 设置协处理器的陷阱门。
	outb_p(inb_p(0x21)&0xfb,0x21);// 允许主8259A 芯片的IRQ2 中断请求。
	outb(inb_p(0xA1)&0xdf,0xA1);// 允许从8259A 芯片的IRQ13 中断请求。
	set_trap_gate(39,&parallel_interrupt);// 设置并行口的陷阱门。
}
```
### blk_dev_init

该函数式块设备初始化函数,什么是块设备？就是可以按照一大块数据进行读取数据的设备，典型的有硬盘，U盘等，与之对比，键盘属于字符设备。
要理解这段函数，我们先来看看request数组吧。
1. request是request结构体类型的数组。
2. request结构体是对一个块设备读取请求的一个抽象，表示了要读取哪个块设备的哪个起始扇区的，要读几个扇区，哪个任务在等待这个请求，
该任务数据的缓冲区和缓冲区头分别是哪个，并且指向了下一个request请求。至于为什么指向下一个请求，我暂时无法回答，猜测与io调度有关系。

分析到这里，基本可以说明blk_dev_init函数在做什么了。初始化request数组而已,以备后用


``` c++
//// 块设备初始化函数，由初始化程序main.c 调用（init/main.c,128）。
// 初始化请求数组，将所有请求项置为空闲项(dev = -1)。有32 项(NR_REQUEST = 32)。
void blk_dev_init (void)
{
	int i;

	for (i = 0; i < NR_REQUEST; i++)
	{
		request[i].dev = -1;
		request[i].next = NULL;
	}
}

struct request
{
  int dev;			/* -1 if no request */// 使用的设备号。
  int cmd;			/* READ or WRITE */// 命令(READ 或WRITE)。
  int errors;			//操作时产生的错误次数。
  unsigned long sector;		// 起始扇区。(1 块=2 扇区)
  unsigned long nr_sectors;	// 读/写扇区数。
  char *buffer;			// 数据缓冲区。
  struct task_struct *waiting;	// 任务等待操作执行完成的地方。
  struct buffer_head *bh;	// 缓冲区头指针(include/linux/fs.h,68)。
  struct request *next;		// 指向下一请求项。
};
```
### chr_dev_init

在Linux0.11源代码中，该函数为空。

### tty_init

该函数大致看了一遍。实现的功能也比较简单。暂时不做进一步分析。

``` c++
//// tty 终端初始化函数。
// 初始化串口终端和控制台终端。
void tty_init (void)
{
	rs_init ();			// 初始化串行中断程序和串行接口1 和2。(serial.c, 37)
	con_init ();			// 初始化控制台终端。(console.c, 617)
}

```

### time_init

该函数设置开机时间

### sched_init 

该函数顾名思义，是调度程序的初始化子程序，我们需要仔细分析一下。首先这需要了解中断描述符表的概念，我的[这篇博客](https://xdushepherd91.github.io/2019/09/24/Linux-0-11-80x86%E4%BF%9D%E6%8A%A4%E6%A8%A1%E5%BC%8F%E5%8F%8A%E5%85%B6%E7%BC%96%E7%A8%8B/)有一个摘抄性质的记录，大家可以进一步参考相关书籍，形成对中断描述符表
的整体认识。

首先来理解一下TSS(Task Status Segment)，任务状态段，该内存段是用来存储一个进程的所有上下文信息的，在进程被切换掉的时候，该段可以保存对应进程的所有的寄存器信息，程序
计数器信息等等，以便下一次该任务被切换执行的时候，可以从被中断的位置开始重新执行。

1. 这就不难理解set_tss_desc的意义了，在全局描述符表中，注册任务0的任务状态段信息，set_ldt_desc则在全局描述符表中注册了任务0的局部描述符表信息。
2. 接下来的初始化全局描述符表的表项，全部设置为0。
3. 将任务0的TSS描述符和局部描述符表分别加载到tr寄存器和ldtr寄存器中。
4. 接下来设置时钟中断程序向量，开始时钟中断。这个很重要，timer_interrupt函数定义在kernel/system_call.s中，是进程调度的入口函数。
5. 最后设置了系统调用的中断程序向量。这里需要理解，我们使用的所有的系统调用，最终都是通过向CPU发出一个中断向量号为0x80的软中断来完成的，至于每个系统调用的参数等，则需要
在进行系统调用的时候存储到寄存器中即可。


``` c++
// 调度程序的初始化子程序。
void sched_init (void)
{
	int i;
	struct desc_struct *p;	// 描述符表结构指针。

	if (sizeof (struct sigaction) != 16)	// sigaction 是存放有关信号状态的结构。
		panic ("Struct sigaction MUST be 16 bytes");
// 设置初始任务（任务0）的任务状态段描述符和局部数据表描述符(include/asm/system.h,65)。
	set_tss_desc (gdt + FIRST_TSS_ENTRY, &(init_task.task.tss));
	set_ldt_desc (gdt + FIRST_LDT_ENTRY, &(init_task.task.ldt));
// 清任务数组和描述符表项（注意i=1 开始，所以初始任务的描述符还在）。
	p = gdt + 2 + FIRST_TSS_ENTRY;
	for (i = 1; i < NR_TASKS; i++)
	{
		task[i] = NULL;
		p->a = p->b = 0;
		p++;
		p->a = p->b = 0;
		p++;
	}
/* 清除标志寄存器中的位NT，这样以后就不会有麻烦 */
// NT 标志用于控制程序的递归调用(Nested Task)。当NT 置位时，那么当前中断任务执行
// iret 指令时就会引起任务切换。NT 指出TSS 中的back_link 字段是否有效。
//  __asm__ ("pushfl ; andl $0xffffbfff,(%esp) ; popfl");	// 复位NT 标志。
	_asm pushfd; _asm and dword ptr ss:[esp],0xffffbfff; _asm popfd;
	ltr (0);			// 将任务0 的TSS 加载到任务寄存器tr。
	lldt (0);			// 将局部描述符表加载到局部描述符表寄存器。
// 注意！！是将GDT 中相应LDT 描述符的选择符加载到ldtr。只明确加载这一次，以后新任务
// LDT 的加载，是CPU 根据TSS 中的LDT 项自动加载。
// 下面代码用于初始化8253 定时器。
	outb_p (0x36, 0x43);		/* binary, mode 3, LSB/MSB, ch 0 */
	outb_p (LATCH & 0xff, 0x40);	/* LSB */// 定时值低字节。
	outb (LATCH >> 8, 0x40);	/* MSB */// 定时值高字节。
  // 设置时钟中断处理程序句柄（设置时钟中断门）。
	set_intr_gate (0x20, &timer_interrupt);
  // 修改中断控制器屏蔽码，允许时钟中断。
	outb (inb_p (0x21) & ~0x01, 0x21);
  // 设置系统调用中断门。
	set_system_gate (0x80, &system_call);
}
```
### buffer_init

buffer_init函数对缓冲区进行初始化，至于缓冲区，前文有专门提到，这里再讲一下，高速缓冲区是用来进行硬盘数据缓冲的，这个区由
缓冲区头和缓冲区块共同组成。缓冲区头是缓冲区块的元数据，用来标记对应缓冲区块是哪个硬盘的哪个扇区的数据缓冲，缓冲区块则存储了
磁盘中多个扇区的完整数据，具体多少个扇区需要根据操作系统的设置来决定，在Linux0.11来说，一个缓冲区块存储了1024字节的数据，而
一个扇区一边是512字节的数据，因此一个缓冲区块存储了两个扇区的数据。

在该函数中：

1. 程序从缓冲区的开头和末端分别开始，初始化一个缓冲区头同时再设置一个缓冲区，并让缓冲区头将缓冲区管理起来，如此循环，知道缓冲区头
和缓冲区相遇结束。
2. 第一个缓冲区头指针为内核代码最末端，保存在start_buffer指针中。
3. 在缓冲区初始化完成后，内核将缓冲区头指针赋予了一个称之为free_list的全局变量之后。
4. 将一个哈希表初始化。

free_list和哈希表存在是为了更加有效的使用缓冲区内存，这个后面的文章会深入分析。


``` c++
//// 缓冲区初始化函数。
// 参数buffer_end 是指定的缓冲区内存的末端。对于系统有16MB 内存，则缓冲区末端设置为4MB。
// 对于系统有8MB 内存，缓冲区末端设置为2MB。
void buffer_init(long buffer_end)
{
	struct buffer_head * h = start_buffer;
	void * b;
	int i;

// 如果缓冲区高端等于1Mb，则由于从640KB-1MB 被显示内存和BIOS 占用，因此实际可用缓冲区内存
// 高端应该是640KB。否则内存高端一定大于1MB。
	if (buffer_end == 1<<20)
		b = (void *) (640*1024);
	else
		b = (void *) buffer_end;
// 这段代码用于初始化缓冲区，建立空闲缓冲区环链表，并获取系统中缓冲块的数目。
// 操作的过程是从缓冲区高端开始划分1K 大小的缓冲块，与此同时在缓冲区低端建立描述该缓冲块
// 的结构buffer_head，并将这些buffer_head 组成双向链表。
// h 是指向缓冲头结构的指针，而h+1 是指向内存地址连续的下一个缓冲头地址，也可以说是指向h
// 缓冲头的末端外。为了保证有足够长度的内存来存储一个缓冲头结构，需要b 所指向的内存块
// 地址>= h 缓冲头的末端，也即要>=h+1。
	while ( (b = (char*)b - BLOCK_SIZE) >= ((void *) (h+1)) ) {
		h->b_dev = 0;			// 使用该缓冲区的设备号。
		h->b_dirt = 0;			// 脏标志，也即缓冲区修改标志。
		h->b_count = 0;			// 该缓冲区引用计数。
		h->b_lock = 0;			// 缓冲区锁定标志。
		h->b_uptodate = 0;		// 缓冲区更新标志（或称数据有效标志）。
		h->b_wait = NULL;		// 指向等待该缓冲区解锁的进程。
		h->b_next = NULL;		// 指向具有相同hash 值的下一个缓冲头。
		h->b_prev = NULL;		// 指向具有相同hash 值的前一个缓冲头。
		h->b_data = (char *) b;	// 指向对应缓冲区数据块（1024 字节）。
		h->b_prev_free = h-1;	// 指向链表中前一项。
		h->b_next_free = h+1;	// 指向链表中下一项。
		h++;					// h 指向下一新缓冲头位置。
		NR_BUFFERS++;			// 缓冲区块数累加。
		if (b == (void *) 0x100000)		// 如果地址b 递减到等于1MB，则跳过384KB，
			b = (void *) 0xA0000;		// 让b 指向地址0xA0000(640KB)处。
	}
	h--;			// 让h 指向最后一个有效缓冲头。
	free_list = start_buffer;		// 让空闲链表头指向头一个缓冲区头。
	free_list->b_prev_free = h;		// 链表头的b_prev_free 指向前一项（即最后一项）。
	h->b_next_free = free_list;		// h 的下一项指针指向第一项，形成一个环链。
	// 初始化hash 表（哈希表、散列表），置表中所有的指针为NULL。
	for (i=0;i<NR_HASH;i++)
		hash_table[i]=NULL;
}	
```
### hd_init

该函数进行了如下操作：
1. 对blk_dev数组中硬盘对应的request_fn进行了赋值，指向了do_hd_request对应的代码。用于处理硬盘数据请求。
2. 设置了硬盘中断向量，并且设置了CPU，允许硬盘控制器发送硬盘中断请求。

后续的文章可以进行深入分析。

``` c++
// 硬盘系统初始化。
void hd_init (void)
{
	blk_dev[MAJOR_NR].request_fn = DEVICE_REQUEST;	// do_hd_request()。
	set_intr_gate (0x2E, &hd_interrupt);	// 设置硬盘中断门向量 int 0x2E(46)。
// hd_interrupt 在(kernel/system_call.s,221)。
	outb_p (inb_p (0x21) & 0xfb, 0x21);	// 复位接联的主8259A int2 的屏蔽位，允许从片
// 发出中断请求信号。
	outb (inb_p (0xA1) & 0xbf, 0xA1);	// 复位硬盘的中断请求屏蔽位（在从片上），允许
// 硬盘控制器发送中断请求信号。
}

```

### floppy_init

软盘和硬盘类似，并且我没有接触过，其代码并没有深入分析。

### sti

开启中断

### move_to_user_mode

切换到用户模式运行。
该函数利用iret 指令实现从内核模式切换到用户模式（初始任务0）。

该段代码，似懂非懂


``` as
#define move_to_user_mode() \
_asm { \
	_asm mov eax,esp /* 保存堆栈指针esp 到eax 寄存器中。*/\
	_asm push 00000017h /* 首先将堆栈段选择符(SS)入栈。*/\
	_asm push eax /* 然后将保存的堆栈指针值(esp)入栈。*/\
	_asm pushfd /* 将标志寄存器(eflags)内容入栈。*/\
	_asm push 0000000fh /* 将内核代码段选择符(cs)入栈。*/\
	_asm push offset l1 /* 将下面标号l1 的偏移地址(eip)入栈。*/\
	_asm iretd /* 执行中断返回指令，则会跳转到下面标号1 处。*/\
_asm l1: mov eax,17h /* 此时开始执行任务0，*/\
	_asm mov ds,ax /* 初始化段寄存器指向本局部表的数据段。*/\
	_asm mov es,ax \
	_asm mov fs,ax \
	_asm mov gs,ax \
}
```
### fork 和 init

在后续的执行中，任务0执行fork系统调用，fork出来的任务1执行init方法，而任务0将会循环执行pause()。


## 总结

本文是我在阅读Linux0.11源码中main函数时的一些总结，有自己的一些理解。还有以下几个方面都有一些遗留问题
1. buffer_init。其中free_list和哈希表的用途需要进一步分析。
2. hd_init。其中do_hd_request函数需要进一步分析
3. move_to_user_mode。该函数的具体意义的进一步分析。













