### 概述

1. 利用setup.s取得的系统参数设置系统的根文件以及一些全局内存变量。
2. 进行所有硬件方面的初始化工作。包括陷阱门、块设备、字符设备、和tty，以及人工设置的第一个任务task 0。
3. 内核初始化完成之后，内核将执行权切换到用户模式。
4. 使用fork(),创建init进程。
5. init函数的功能：
   1. 安装根文件系统
   2. 显示系统信息
   3. 运行系统初始化资源配置文件rc的命令
   4. 执行用户登录shell程序
### trap_init() 函数

异常陷阱初始化子程序。设置其中断调用门。该方法调用了如下两类方法:

```c
void trap_init(void)
{
	int i;
    set_trap_gate(0,&divide_error);
    set_system_gate(3,&int3);
}

```
#### set_trap_gate函数

```c
#define set_trap_gate(n,addr) _set_gate((unsigned long*)(&(idt[n])),15,0,(unsigned long)addr)
```

#### set_system_gate函数

```c
#define set_system_gate(n,addr) _set_gate((unsigned long*)(&(idt[n])),15,3,(unsigned long)addr)
```

#### _set_gate函数

可以看到，_set_gate函数的入参分别是，描述符的地址，描述符的类型，描述符的特权值，以及处理程序的地址。
1. 地址addr对应的是set_trap_gate(0,&divide_error)调用中的&divide_error地址。
2. type和dpl对应了门描述符的类型和特权的位值。
3. 描述符地址。我们可以看到，描述符地址是&(idt[n])，idt，我们可以联想到idt，中断描述符表。我们在继续找一下这个idt数组。idt在head.h中定义，同样的还有gdt，基本证实了我们的推断，extern desc_table idt, gdt。他们是一个desc_struct类型。存疑问？idt，gdt的地址如何确定。

总结，至此，我们可以确定，_set_gate函数的作用是将中断处理函数注册到idt表中去。
更进一步地将，set_trap_gate,set_system_gate等函数分别将陷入门，系统门等程序注册到了idt表中。


```c
//// 设置门描述符宏函数。
// 参数：gate_addr -描述符地址；type -描述符中类型域值；dpl -描述符特权层值；addr -偏移地址。
// %0 - (由dpl,type 组合成的类型标志字)；%1 - (描述符低4 字节地址)；
// %2 - (描述符高4 字节地址)；%3 - edx(程序偏移地址addr)；%4 - eax(高字中含有段选择符)。
void _inline _set_gate(unsigned long *gate_addr, \
					   unsigned short type, \
					   unsigned short dpl, \
					   unsigned long addr) 
{// c语句和汇编语句都可以通过
	gate_addr[0] = 0x00080000 + (addr & 0xffff);
	gate_addr[1] = 0x8000 + (dpl << 13) + (type << 8) + (addr & 0xffff0000);
/*	unsigned short tmp = 0x8000 + (dpl << 13) + (type << 8);
	_asm mov eax,00080000h ;
	_asm mov edx,addr ;
	_asm mov ax,dx ;// 将偏移地址低字与段选择符组合成描述符低4 字节(eax)。
	_asm mov dx,tmp ;// 将类型标志字与偏移高字组合成描述符高4 字节(edx)。
	_asm mov ebx,gate_addr
	_asm mov [ebx],eax ;// 分别设置门描述符的低4 字节和高4 字节。
	_asm mov [ebx+4],edx ;*/
}
```

#### trap_init(void)总结

经过上述的代码分析，我们可以确定trap_init向idt中注册了多个陷入处理函数。


### mem_init(long start_mem, long end_mem)函数

该函数根据入参将内存的高位地址记录到HIGH_MEMORY变量中，并且初始化了全局变量
mem_map，该变量代表了主内存中每4k页面的当前状态

```c
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

### blk_dev_init (void)函数

request结构体中，包含了将nr扇区数据加载到内存中需要的所有信息，比如，设备号，读命令或者写命令，起始扇区，需要读写的扇区数量，缓冲区地址，进程结构体地址，缓冲区头指针，下一个请求等等。

blk_dev_init函数的主要作用是初始化32个request备用。

```c 
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

struct request request[NR_REQUEST] = {0};

/*
* OK，下面是request 结构的一个扩展形式，因而当实现以后，我们就可以在分页请求中
* 使用同样的request 结构。在分页处理中，'bh'是NULL，而'waiting'则用于等待读/写的完成。
*/
// 下面是请求队列中项的结构。其中如果dev=-1，则表示该项没有被使用。
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
### tty_init函数

该函数的作用如下

```c
//// tty 终端初始化函数。
// 初始化串口终端和控制台终端。
void tty_init (void)
{
	rs_init ();			// 初始化串行中断程序和串行接口1 和2。(serial.c, 37)
	con_init ();			// 初始化控制台终端。(console.c, 617)
}
```

#### rs_init函数

1. 为串口1和串口2设置中断门向量。
2. 初始化串行口。
3. 允许主8259A 芯片的中断。

```c
//// 初始化串行中断程序和串行接口。
void
rs_init (void)
{
	set_intr_gate (0x24, rs1_interrupt);	// 设置串行口1 的中断门向量(硬件IRQ4 信号)。
	set_intr_gate (0x23, rs2_interrupt);	// 设置串行口2 的中断门向量(硬件IRQ3 信号)。
	init (tty_table[1].read_q.data);	// 初始化串行口1(.data 是端口号)。
	init (tty_table[2].read_q.data);	// 初始化串行口2。
	outb (inb_p (0x21) & 0xE7, 0x21);	// 允许主8259A 芯片的IRQ3，IRQ4 中断信号请求。
}
```
#### con_init函数

对控制台进行初始化操作


