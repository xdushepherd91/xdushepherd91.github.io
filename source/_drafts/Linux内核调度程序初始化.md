### sched_init函数

1. 检查sigaction的内存结构
2. 设置任务状态段描述符
3. 设置任务局部描述表描述符
4. 清任务数组和描述符表项（注意i=1 开始，所以初始任务的描述符还在）。
5. 清除标志寄存器中的位NT，这样以后就不会有麻烦。  NT 标志用于控制程序的递归调用(Nested Task)。当NT 置位时，那么当前中断任务执行，iret 指令时就会引起任务切换。NT 指出TSS 中的back_link 字段是否有效。
6. 将任务0 的TSS 加载到任务寄存器tr。
7. // 将局部描述符表加载到局部描述符表寄存器。
// 注意！！是将GDT 中相应LDT 描述符的选择符加载到ldtr。只明确加载这一次，以后新任务
// LDT 的加载，是CPU 根据TSS 中的LDT 项自动加载。
8. 初始化8253 定时器。
10. 设置时钟中断处理程序句柄（设置时钟中断门）。
11. 修改中断控制器屏蔽码，允许时钟中断。
12. 设置系统调用中断门。


### buffer_init(long buffer_end)函数

缓冲区初始化函数。
参数buffer_end 是指定的缓冲区内存的末端。对于系统有16MB 内存，则缓冲区末端设置为4MB。
对于系统有8MB 内存，缓冲区末端设置为2MB。
该函数的作用是，建立空闲缓冲区环链表，并获取系统中缓冲块的数目。
1. 缓冲区高端开始划分1K 大小的缓冲块
2. 与此同时在缓冲区低端建立描述该缓冲块的结构buffer_head，并将这些buffer_head 组成双向链表
3. h 是指向缓冲头结构的指针，而h+1 是指向内存地址连续的下一个缓冲头地址
4. 缓冲头结构形成环形链表
5. 初始化hash 表（哈希表、散列表），置表中所有的指针为NULL。



```c
// 缓冲区头数据结构。（极为重要！！！）
// 在程序中常用bh 来表示buffer_head 类型的缩写。
struct buffer_head
{
  char *b_data;			/* pointer to data block (1024 bytes) *///指针。
  unsigned long b_blocknr;	/* block number */// 块号。
  unsigned short b_dev;		/* device (0 = free) */// 数据源的设备号。
  unsigned char b_uptodate;	// 更新标志：表示数据是否已更新。
  unsigned char b_dirt;		/* 0-clean,1-dirty *///修改标志:0 未修改,1 已修改.
  unsigned char b_count;	/* users using this block */// 使用的用户数。
  unsigned char b_lock;		/* 0 - ok, 1 -locked */// 缓冲区是否被锁定。
  struct task_struct *b_wait;	// 指向等待该缓冲区解锁的任务。
  struct buffer_head *b_prev;	// hash 队列上前一块（这四个指针用于缓冲区的管理）。
  struct buffer_head *b_next;	// hash 队列上下一块。
  struct buffer_head *b_prev_free;	// 空闲表上前一块。
  struct buffer_head *b_next_free;	// 空闲表上下一块。
};
```

### void hd_init (void)函数

硬盘初始化函数。
1. 设置硬盘请求操作的函数指针。
2. 设置硬盘中断门向量。
2. 允许硬盘控制器发送中断请求信号。


```c
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

/* blk_dev_struct is:
* do_request-address
* next-request
*/
/* blk_dev_struct 块设备结构是：(kernel/blk_drv/blk.h,23)
* do_request-address //对应主设备号的请求处理程序指针。
* current-request // 该设备的下一个请求。
*/
// 该数组使用主设备号作为索引（下标）。
struct blk_dev_struct blk_dev[NR_BLK_DEV] = {
	{NULL, NULL},		/* no_dev */// 0 - 无设备。
	{NULL, NULL},		/* dev mem */// 1 - 内存。
	{NULL, NULL},		/* dev fd */// 2 - 软驱设备。
	{NULL, NULL},		/* dev hd */// 3 - 硬盘设备。
	{NULL, NULL},		/* dev ttyx */// 4 - ttyx 设备。
	{NULL, NULL},		/* dev tty */// 5 - tty 设备。
	{NULL, NULL}		/* dev lp */// 6 - lp 打印机设备。
};
```
#### 硬盘的读写请求操作

```c

// 执行硬盘读写请求操作。
void do_hd_request (void)
{
	int i, r;
	unsigned int block, dev;
	unsigned int sec, head, cyl;
	unsigned int nsect;

	INIT_REQUEST;		// 检测请求项的合法性(参见kernel/blk_drv/blk.h,127)。
// 取设备号中的子设备号(见列表后对硬盘设备号的说明)。子设备号即是硬盘上的分区号。
	dev = MINOR (CURRENT->dev);	// CURRENT 定义为(blk_dev[MAJOR_NR].current_request)。
	block = CURRENT->sector;	// 请求的起始扇区。
// 如果子设备号不存在或者起始扇区大于该分区扇区数-2，则结束该请求，并跳转到标号repeat 处
// （定义在INIT_REQUEST 开始处）。因为一次要求读写2 个扇区（512*2 字节），所以请求的扇区号
// 不能大于分区中最后倒数第二个扇区号。
	if (dev >= 5 * NR_HD || block + 2 > hd[dev].nr_sects)
	{
		end_request (0);
		goto repeat;		// 该标号在blk.h 最后面。
	}
	block += hd[dev].start_sect;	// 将所需读的块对应到整个硬盘上的绝对扇区号。
	dev /= 5;			// 此时dev 代表硬盘号（0 或1）。
// 下面嵌入汇编代码用来从硬盘信息结构中根据起始扇区号和每磁道扇区数计算在磁道中的
// 扇区号(sec)、所在柱面号(cyl)和磁头号(head)。
	sec = hd_info[dev].sect;
	_asm {
		mov eax,block
		xor edx,edx
		mov ebx,sec
		div ebx
		mov block,eax
		mov sec,edx
	}
//__asm__ ("divl %4": "=a" (block), "=d" (sec):"" (block), "1" (0),
//	   "r" (hd_info[dev].
//		sect));
	head = hd_info[dev].head;
	_asm {
		mov eax,block
		xor edx,edx
		mov ebx,head
		div ebx
		mov cyl,eax
		mov head,edx
	}
//__asm__ ("divl %4": "=a" (cyl), "=d" (head):"" (block), "1" (0),
//	   "r" (hd_info[dev].
//		head));
	sec++;
	nsect = CURRENT->nr_sectors;	// 欲读/写的扇区数。
// 如果reset 置1，则执行复位操作。复位硬盘和控制器，并置需要重新校正标志，返回。
	if (reset)
	{
		reset = 0;
		recalibrate = 1;
		reset_hd (CURRENT_DEV);
		return;
	}
// 如果重新校正标志(recalibrate)置位，则首先复位该标志，然后向硬盘控制器发送重新校正命令。
	if (recalibrate)
	{
		recalibrate = 0;
		hd_out (dev, hd_info[CURRENT_DEV].sect, 0, 0, 0,
		  WIN_RESTORE, &recal_intr);
		return;
	}
// 如果当前请求是写扇区操作，则发送写命令，循环读取状态寄存器信息并判断请求服务标志
// DRQ_STAT 是否置位。DRQ_STAT 是硬盘状态寄存器的请求服务位（include/linux/hdreg.h，27）。
	if (CURRENT->cmd == WRITE)
	{
		hd_out (dev, nsect, sec, head, cyl, WIN_WRITE, &write_intr);
		for (i = 0; i < 3000 && !(r = inb_p (HD_STATUS) & DRQ_STAT); i++)
// 如果请求服务位置位则退出循环。若等到循环结束也没有置位，则此次写硬盘操作失败，去处理
// 下一个硬盘请求。否则向硬盘控制器数据寄存器端口HD_DATA 写入1 个扇区的数据。
		if (!r)
		{
			bad_rw_intr ();
			goto repeat;		// 该标号在blk.h 最后面，也即跳到301 行。
		}
		port_write (HD_DATA, CURRENT->buffer, 256);
// 如果当前请求是读硬盘扇区，则向硬盘控制器发送读扇区命令。
	}
	else if (CURRENT->cmd == READ)
	{
		hd_out (dev, nsect, sec, head, cyl, WIN_READ, &read_intr);
	}
	else
		panic ("unknown hd-command");
}

```

#### hd_out

```c
static void hd_out (unsigned int drive, unsigned int nsect, unsigned int sect,
		    unsigned int head, unsigned int cyl, unsigned int cmd,
		    void (*intr_addr) (void))
{
	register int port; //asm ("dx");	// port 变量对应寄存器dx。

	if (drive > 1 || head > 15)	// 如果驱动器号(0,1)>1 或磁头号>15，则程序不支持。
		panic ("Trying to write bad sector");
	if (!controller_ready ())	// 如果等待一段时间后仍未就绪则出错，死机。
		panic ("HD controller not ready");
	do_hd = intr_addr;		// do_hd 函数指针将在硬盘中断程序中被调用。
	outb_p (hd_info[drive].ctl, HD_CMD);	// 向控制寄存器(0x3f6)输出控制字节。
	port = HD_DATA;		// 置dx 为数据寄存器端口(0x1f0)。
	outb_p (hd_info[drive].wpcom >> 2, ++port);	// 参数：写预补偿柱面号(需除4)。
	outb_p (nsect, ++port);	// 参数：读/写扇区总数。
	outb_p (sect, ++port);	// 参数：起始扇区。
	outb_p (cyl, ++port);		// 参数：柱面号低8 位。
	outb_p (cyl >> 8, ++port);	// 参数：柱面号高8 位。
	outb_p (0xA0 | (drive << 4) | head, ++port);	// 参数：驱动器号+磁头号。
	outb (cmd, ++port);		// 命令：硬盘控制命令。
}
```
#### read_intr

```c
//// 读操作中断调用函数。将在执行硬盘中断处理程序中被调用。
static void read_intr (void)
{
	if (win_result ())
	{				// 若控制器忙、读写错或命令执行错，
		bad_rw_intr ();		// 则进行读写硬盘失败处理
		do_hd_request ();		// 然后再次请求硬盘作相应(复位)处理。
		return;
	}
	port_read (HD_DATA, CURRENT->buffer, 256);	// 将数据从数据寄存器口读到请求结构缓冲区。
	CURRENT->errors = 0;		// 清出错次数。
	CURRENT->buffer += 512;	// 调整缓冲区指针，指向新的空区。
	CURRENT->sector++;		// 起始扇区号加1，
	if (--CURRENT->nr_sectors)
	{				// 如果所需读出的扇区数还没有读完，则
		do_hd = &read_intr;	// 再次置硬盘调用C 函数指针为read_intr()
		return;			// 因为硬盘中断处理程序每次调用do_hd 时
	}				// 都会将该函数指针置空。参见system_call.s
	end_request (1);		// 若全部扇区数据已经读完，则处理请求结束事宜，
	do_hd_request ();		// 执行其它硬盘请求操作。
}
```
### floppy_init

与硬盘初始化类似