---
title: java并发包基础类AbstractQueuedSynchronizer源码分析
tags:
  - jdk
categories:
  - java
date: 2019-11-28 17:55:02
---

### 概述

AbstractQueuedSynchronizer代码之前读过，没有写个总结，这次在看线程池源码的时候又遇到了，发现忘记了很多关键性的内容，因此，特地翻出来，再看看

<!-- more -->

### 组件

#### Node

AQS的内部类，其核心变量有：
1. waitStatus
2. prev
3. next
4. thread
5. nextWaiter

