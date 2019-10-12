---
title: Linux-0.11内核main函数执行过程分析——2
tags: 
- linux
- 底层
categories: 
- Linux 内核
date: 2019-10-12 14:22:31
---

## 概述

上一篇涉及到了main函数在fork任务1之前的一些行为，由于任务1执行的init函数涉及内容较多，因此将其内容移动
到本篇来继续分析。

``` c++
void init(void)
{
	int pid,i;

// 读取硬盘参数包括分区表信息并建立虚拟盘和安装根文件系统设备。
// 该函数是在25 行上的宏定义的，对应函数是sys_setup()，在kernel/blk_drv/hd.c。
	setup((void *) &drive_info);

	(void) open("/dev/tty0",O_RDWR,0);	// 用读写访问方式打开设备“/dev/tty0”，
										// 这里对应终端控制台。
										// 返回的句柄号0 -- stdin 标准输入设备。
	(void) dup(0);		// 复制句柄，产生句柄1 号-- stdout 标准输出设备。
	(void) dup(0);		// 复制句柄，产生句柄2 号-- stderr 标准出错输出设备。
	printf("%d buffers = %d bytes buffer space\n\r",NR_BUFFERS, \
		NR_BUFFERS*BLOCK_SIZE);	// 打印缓冲区块数和总字节数，每块1024 字节。
	printf("Free mem: %d bytes\n\r",memory_end-main_memory_start);//空闲内存字节数。

// 下面fork()用于创建一个子进程(子任务)。对于被创建的子进程，fork()将返回0 值，
// 对于原(父进程)将返回子进程的进程号。所以if (!(pid=fork())) {...} 内是子进程执行的内容。
// 该子进程关闭了句柄0(stdin)，以只读方式打开/etc/rc 文件，并执行/bin/sh 程序，所带参数和
// 环境变量分别由argv_rc 和envp_rc 数组给出。参见后面的描述。
	if (!(pid=fork())) {
		close(0);
		if (open("/etc/rc",O_RDONLY,0))
			_exit(1);	// 如果打开文件失败，则退出(/lib/_exit.c)。
		execve("/bin/sh",argv_rc,envp_rc);	// 装入/bin/sh 程序并执行。(/lib/execve.c)
		_exit(2);	// 若execve()执行失败则退出(出错码2,“文件或目录不存在”)。
	}

// 下面是父进程执行的语句。wait()是等待子进程停止或终止，其返回值应是子进程的
// 进程号(pid)。这三句的作用是父进程等待子进程的结束。&i 是存放返回状态信息的
// 位置。如果wait()返回值不等于子进程号，则继续等待。
	if (pid>0)
		while (pid != wait(&i))
		{	/* nothing */;}

// --
// 如果执行到这里，说明刚创建的子进程的执行已停止或终止了。下面循环中首先再创建
// 一个子进程，如果出错，则显示“初始化程序创建子进程失败”的信息并继续执行。对
// 于所创建的子进程关闭所有以前还遗留的句柄(stdin, stdout, stderr)，新创建一个
// 会话并设置进程组号，然后重新打开/dev/tty0 作为stdin，并复制成stdout 和stderr。
// 再次执行系统解释程序/bin/sh。但这次执行所选用的参数和环境数组另选了一套（见上面）。
// 然后父进程再次运行wait()等待。如果子进程又停止了执行，则在标准输出上显示出错信息
//		“子进程pid 停止了运行，返回码是i”，
// 然后继续重试下去…，形成“大”死循环。
	while (1) {
		if ((pid=fork())<0) {
			printf("Fork failed in init\r\n");
			continue;
		}
		if (!pid) {
			close(0);close(1);close(2);
			setsid();
			(void) open("/dev/tty0",O_RDWR,0);
			(void) dup(0);
			(void) dup(0);
			_exit(execve("/bin/sh",argv,envp));
		}
		while (1)
			if (pid == wait(&i))
				break;
		printf("\n\rchild %d died with code %04x\n\r",pid,i);
		sync();
	}
	_exit(0);	/* NOTE! _exit, not exit() */
}
```

### setup

该函数对应函数是sys_setup()，在kernel/blk_drv/hd.c。代码不在这里贴了。讲一下这个函数的内容：

1. 在全局变量hd数组中设置每个硬盘的起始扇区号和总的扇区数。
2. 读取第一个硬盘的第一个扇区，获取其中的分区表信息。
3. rd_load函数执行。加载（创建）RAMDISK(kernel/blk_drv/ramdisk.c,71)。
4. mount_root函数执行。// 安装根文件系统(fs/super.c,242)。

#### mount_root

由于对Linux中文件系统的挂载有一个深入的接触，这里来进一步分析一下mount_root函数的调用过程吧。

在分析函数之前，先来明确一下Linux中文件系统的一些概念：

1. super_block。超级块。超级块用来表示一个文件系统，其内容主要包括了该文件系统的inode数目，逻辑块数，
inode位图所占用的数据块数，逻辑块位图所占用的的数据块数，第一个数据块等等。那么什么优势inode呢？
2. inode。一般我们就称之为inode，不翻译。 一个inode通常来说代表了一个文件，当然也可以代表一个目录。其主要
内容包括文件的类型和属性，用户id，文件大小，修改时间，指向该inode的链接数以及最重要的，文件的数据块标志数组，
包括7个直接，1个间接和两个双重间接。
3. 对于上述两个数据结构，在硬盘中必然都有弃数据结构，同时，其在内存中的数据结构相比有硬盘数据结构都有了扩充。
因此Linux0.11中，分别定义了d_inode,m_inode,d_super_block和super_block四种数据结构来分别表示。
4. 对于目录，我们还需要了解数据结构dir_entry。该数据结构表示了一个目录。
5. 以上数据结构的定义位于Linux0.11源代码的include/linux/fs.h文件中。

总结，Linux文件系统中，我们将位于磁盘上的文件系统超级块数据读取到内存中并表示为super_block。我们通过super block
中的inode位图可以知道哪些inode还可以使用，哪些逻辑块还可以使用，这样我们在新建文件的时候就可以通过查询super block
来找到可用的inode和可用的数据块，并修改super block中的位图信息。并在合适的时候将超级块的修改信息回写到硬盘中

接下来我们来列举mount_root函数所做的事情

1. 初始化file_table全局数组。该数组大小为64，说明Linux0.11同时只能打开64个文件。
2. 初始化超级块数组。超级块数组大小为8，说明Linux0.11最多只能挂载8个文件系统。
3. 从根设备上读取超级块信息。
4. 读取根设备的第一个inode到内存中。
5. 设置该inode为mount点和根目录挂载点。
6. 设置当前进程的工作目录和根节点为第一个inode。
7. 进行一些文件系统数据的计算，并打印到控制台输出。

### open

系统调用open将/dev/tty0打开设置为标准输入0，使用dup系统调用将其赋值为
标准输出1,和标准输出2。

### fork及子程序执行

init函数在调用fork之后，在任务1中使用wait系统调用进行等待，在子任务中执行以下操作：

1. 关闭标准输入0.
2. 以只读方式打开/etc/rc.
3. 使用execve系统调用执行/bin/sh程序。


###  最后一步

init进程执行到这里的时候，就进入了循环，这就是Linux系统的常态，打开一个/bin/sh程序，开始执行，
当执行的sh程序推出之后，同步文件系统，再打开一个shell程序，如此往复。

``` c++
	while (1) {
		if ((pid=fork())<0) {
			printf("Fork failed in init\r\n");
			continue;
		}
		if (!pid) {
			close(0);close(1);close(2);
			setsid();
			(void) open("/dev/tty0",O_RDWR,0);
			(void) dup(0);
			(void) dup(0);
			_exit(execve("/bin/sh",argv,envp));
		}
		while (1)
			if (pid == wait(&i))
				break;
		printf("\n\rchild %d died with code %04x\n\r",pid,i);
		sync();
	}
```

## 总结


本文对init进程的行为进行了一个较为深入的分析，但是还存在以下一些系统调用需要进行进一步的分析：

1. fork
2. execve
3. open
4. close

这些系统调用中后续需要对fork和execve进行分析。











