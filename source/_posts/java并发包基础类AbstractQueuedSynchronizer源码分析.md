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


### acquire

先来看源码吧

````java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
````

tryAcquire方法由子类实现，这里先不做过多的解释。先来看addWaiter方法。

#### addWaiter方法

````java
  private Node addWaiter(Node mode) {
      Node node = new Node(Thread.currentThread(), mode);
      // Try the fast path of enq; backup to full enq on failure
      Node pred = tail;
      if (pred != null) {
          node.prev = pred;
          if (compareAndSetTail(pred, node)) {
              pred.next = node;
              return node;
          }
      }
      enq(node);
      return node;
  }
````

addWaiter的逻辑如下：
1. 新建一个Node实例。
2. 尝试将node实例加入到node队列中去
   1. 使用cas方法，希望快速地把node加入到队列中去，若成功，直接返回node。
   2. cas失败，使用enq方法将其入队。之后返回enq方法。成功后返回node实例。

##### enq方法

````java
  private Node enq(final Node node) {
      for (;;) {
          Node t = tail;
          if (t == null) { // Must initialize
              if (compareAndSetHead(new Node()))
                  tail = head;
          } else {
              node.prev = t;
              if (compareAndSetTail(t, node)) {
                  t.next = node;
                  return t;
              }
          }
      }
  }
````

enq方法的主要目的就是将其加入到队列中去，那么我们来看看其主要逻辑：
1. 在循环中，获取到当前的tail节点，如果t==null，说明当前的没有等待队列，那么我们先使用cas初始化一个空的node同时作为头结点和尾节点
2. 否则，另当前的node的prev指针指向tail节点，并使用使用cas将当前节点设置为为节点，成功，则将之前为节点的next指针指向当前节点，并返回之前的tail节点。
3. 否则，进入下一次循环继续重试。


#### acquireQueued方法

```` java
  final boolean acquireQueued(final Node node, int arg) {
      boolean failed = true;
      try {
          boolean interrupted = false;
          for (;;) {
              final Node p = node.predecessor();
              if (p == head && tryAcquire(arg)) {
                  setHead(node);
                  p.next = null; // help GC
                  failed = false;
                  return interrupted;
              }
              if (shouldParkAfterFailedAcquire(p, node) &&
                  parkAndCheckInterrupt())
                  interrupted = true;
          }
      } finally {
          if (failed)
              cancelAcquire(node);
      }
  }
````
这里写一下上面这段代码的逻辑，首先，我们先要指出，一般情况下，执行到该方法时，是一个线程正在执行lock.lock()方法，那么，函数中的循环也是在该线程中运行的，这里要明白这个大前提，否则就会云里雾里。

在循环中，执行逻辑如下：
1. 获取当前节点的上一个节点。
2. 如果前一个结点时节点队列的头节点，并且使用tryAcquire方法获取到了锁，那么我们可以保证其他的线程得不到锁了，我们把当前节点设置未新的头节点，然后返回在我们等待锁的过程中，是否被中断过，如果中断过，返回中断未true，否则未false
3. 假设在2中没有获取到锁，那么我们就要遇到两个函数shouldParkAfterFailedAcquire,parkAndCheckInterrupt,其内容马上会详细分析，总的来说，这两个方法的目的时，判断当前线程是否可以被暂停，可以暂停的话，将其暂停，并检查在暂停期间是否被interrupt过，如果时的，那么interrupted在当前线程再次被运行时，会设置为ture。

下面我们分别分析shouldParkAfterFailedAcquire方法和parkAndCheckInterrupt方法

##### shouldParkAfterFailedAcquire方法

这里特意从Node中提取出来了几个waitStatus可能值，可以看到，
1. 如果前一个节点的ws时signal的，那么直接返回true说明，当前进程可以被暂停。
2. 如果ws大于0，也就是CANCELLED状态，那么我们要通过一个循环，向前找到第一个状态部位CANCELLED的node，并使其指向当前node。
3. 如果ws是其他的值，那么使用cas尝试更改其前一个节点waitstatus
4. 返回false。

````java
  /** waitStatus value to indicate thread has cancelled */
  static final int CANCELLED =  1;
  /** waitStatus value to indicate successor's thread needs unparking */
  static final int SIGNAL    = -1;
  /** waitStatus value to indicate thread is waiting on condition */
  static final int CONDITION = -2;
  /**
    * waitStatus value to indicate the next acquireShared should
    * unconditionally propagate
    */
  static final int PROPAGATE = -3;


  private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
      int ws = pred.waitStatus;
      if (ws == Node.SIGNAL)
          /*
            * This node has already set status asking a release
            * to signal it, so it can safely park.
            */
          return true;
      if (ws > 0) {
          /*
            * Predecessor was cancelled. Skip over predecessors and
            * indicate retry.
            */
          do {
              node.prev = pred = pred.prev;
          } while (pred.waitStatus > 0);
          pred.next = node;
      } else {
          /*
            * waitStatus must be 0 or PROPAGATE.  Indicate that we
            * need a signal, but don't park yet.  Caller will need to
            * retry to make sure it cannot acquire before parking.
            */
          compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
      }
      return false;
  }
````

##### parkAndCheckInterrupt方法

逻辑比较简单，当当前线程可以被暂停的时候，那么
1. 使用LockSupport.park方法，将当前线程暂停
2. 在暂停回来的时候，返回当前线程在暂停时，是否接收到了中断请求。

```` java
  private final boolean parkAndCheckInterrupt() {
      LockSupport.park(this);
      return Thread.interrupted();
  }
````

#### selfInterrupt方法

如果当前线程在执行过程中被中断过，那么我们在获取到锁之后，自行再次中断一次。

````java
    static void selfInterrupt() {
        Thread.currentThread().interrupt();
    }
````


#### 小结

以上为acquire方法的执行逻辑。简单总结:
1. tryAcquire方法调用返回true，说明直接获取到了锁
2. 否则，尝试将将当前线程加入队列并在acquireQueued方法中一直尝试获取锁
3. 判断是否需要执行中断操作。


### release方法

在acquire方法中，其实我们对waitStatus的意义并不是很明了的，那么我们试试在release方法中是否可以找到一些答案？

````java
  public final boolean release(int arg) {
      if (tryRelease(arg)) {
          Node h = head;
          if (h != null && h.waitStatus != 0)
              unparkSuccessor(h);
          return true;
      }
      return false;
  }
````
首先，tryRelase方法是子类实现的，这里先不关注。我们先来看看unparkSuccessor方法吧

#### unparkSuccessor方法

对于该方法，需要和前面的parkAndCheckInterrupt方法对应起来，我们现在要做的事情就是将之前被park的线程unpark，使其重新进入线程的调度队列中。具体逻辑
1. 更新当前线程对应的node的waitStatus状态
2. 寻找下一个waitStatus未被cancelled的的node
3. 使用LockSupport.unpark将2中找到的node重新放入线程调度队列中 

````java
  private void unparkSuccessor(Node node) {
      int ws = node.waitStatus;
      if (ws < 0)
          compareAndSetWaitStatus(node, ws, 0);

      Node s = node.next;
      if (s == null || s.waitStatus > 0) {
          s = null;
          //找到node的next
          for (Node t = tail; t != null && t != node; t = t.prev)
              if (t.waitStatus <= 0)
                  s = t;
      }
      if (s != null)
          LockSupport.unpark(s.thread);
  }
````









