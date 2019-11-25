---
title: Linux0.11内核中fork系统调用实现代码分析
tags:
  - null
categories:
  - null
date: 2019-10-15 17:06:39
---


### 暂时记住

1. 在Linux0.11版本中，代码段和数据段的基地址是相等的
2. 当新建一个进程的时候，其代码段和数据段的基地址为nr * 0x4000000，其中
nr为任务号，这就保证了不同任务之间的内存隔离。
3. 


### copy_page_tables

