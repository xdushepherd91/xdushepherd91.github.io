---
title: Linux-0.11-微型计算机组成结构笔记
date: 2019-09-24 10:22:10
tags:
---

### 概览

CPU通过地址线，数据线和控制信号线组成的本地总线与系统的其他部分进行数据通信。其中，
1. 地址总线，用于提供内存或者io设备的地址，指明需要读写数据的具体位置。
2. 数据总线，用于在CPU和内存或者io设备之间提供数据传输的通道。
3. 控制总线，用于指挥执行的具体读写操作。

在80386内部，其地址总线和数据总线都是32位的。其可寻址的地址空间范围是2sup32，从0到4GB。

下图中的控制器都集成在了pc主板上，控制卡（或者称为适配器）则通过扩展卡槽与系统总线相连接。
git a
![test](https://raw.githubusercontent.com/xdushepherd91/xdushepherd91.github.io/master/pc-structure.png)

### 现代pc

现代pc的主板主要使用2个超大规模芯片构成的芯片组或者芯片集组成：北桥芯片和南桥芯片。

1. 北桥。用于CPU、内存、视频接口，具有很高的传输速率。
2. 南桥。用来管理低速、中速组件。

![test](https://raw.githubusercontent.com/xdushepherd91/xdushepherd91.github.io/master/modern-pc-structure.png)


### io寻址方式和访问控制方式

CPU为了访问io接口控制器或者控制卡上的数据和状态信息，需要首先指定他们的地址。这种地址就成为IO端口地址或者简称端口。端口地址的设置方法有两种：统一编址和独立编址。

1. 统一编址。将IO控制器中的端口地址归入存储器寻址空间范围内。
2. 独立编址。单独对待IO控制器和控制卡的地址，使用专门的IO指令来访问端口

IBM兼容pc主要使用独立编址方式。采用独立的IO地址空间对控制设备的寄存器进行寻址和访问。

IBM兼容pc也部分地使用了统一编址的方式。例如，CGA显示卡显示内存的地址就直接占用存储器地址的0xB800-0xBC00范围，若要让一个字符显示在屏幕上，可以直接使用内存操作指令往这个内存区域执行写操作。

### 接口访问控制

pc机的io接口数据访问控制方式一般可采用：
1. 程序循环查询方式。浪费CPU时间。
2. 中断处理方式。需要中断控制器支持。
3. DMA传输方式。需要DMA控制器支持。

目前Linux操作系统主要使用了后两种方式。


### 主存储器、BIOS和CMOS存储器

#### 主存储器

32为cpu的寻址范围已经达到了4GB，而ROM中的BIOS一直处于CPU能寻址的内存最高端位置处。

#### 基本输出输出程序BIOS

存放在ROM中的系统BIOS程序主要用于计算机开机时执行的系统各部分自检，建立起操作系统需要使用的各种配置表，例如中断向量表，硬盘参数表等。并把处理器和系统其余部分初始化到一个已知状态，为操作系统提供硬件设备接口服务。

在机器上电之后，CPU会自动把代码段寄存器cs设置为0xF000，其段基地址则被设置为0xFFFFFFF0，段长度设置为64KB。而IP则被设置为0xFFF0，因此此时CPU代码指针指向0xFFFFFFF0处，即4G空间的最后64K的最后16字节处。这正是系统ROM BIOS程序的存放位置。 并且BIOS会在这里存放一条jmp指令，使得cpu跳转到最后64K范围内的某一条指令开始执行。

#### CMOS存储器

CMOS用来存放系统的实时时钟信息和系统硬件配置信息，通常和实时时钟芯片集成，使用io指令来访问。

### 控制器和控制卡

#### 中断控制器

80386中的中断控制器用于实现io设备的中断控制存取方式，并且能为15个设备提供独立的中断控制功能。

在计算机初始化的时候，CPU在内存0x000——0xFFF区域建立一个中断向量表。在内核初始化的阶段，Linux又重新对其进行了设置。
![](https://raw.githubusercontent.com/xdushepherd91/xdushepherd91.github.io/master/inter.png)

当一个pc计算机上电的时候，上图中的中断请求号会被ROM BIOS设置成下表中列出的中断向量号。Linux系统不直接使用该中断向量号，在内核初始化的时候，内核会重新设置对应。

![](https://raw.githubusercontent.com/xdushepherd91/xdushepherd91.github.io/master/inter-table.png)


当一个pc计算机上电的时候，上图中的中断请求号会被ROM BIOS设置成下表中列出的中断向量号。Linux系统不直接使用该中断向量号，在内核初始化的时候，内核会重新设置对应。

![](https://raw.githubusercontent.com/xdushepherd91/xdushepherd91.github.io/master/inter-table.png)

#### DMA控制器

DMA控制器的主要功能是让外部设备直接与内存进行数据传输来增强系统的性能，有了DMA控制器，外设和内存能在不受CPU控制的条件下进行传输。

#### 定时器、计数器

定时器按照一定的时间间隔发送中断请求信号。中断请求时Linux内核工作的脉搏，用于定时切换当前执行的任务和统计每个任务使用的系统时间资源量。

#### 键盘控制器

接受键盘输入请求。

#### 串行控制卡

#### 显示控制

#### 软盘和硬盘控制器

![](https://raw.githubusercontent.com/xdushepherd91/xdushepherd91.github.io/master/disk-structure.png)


























