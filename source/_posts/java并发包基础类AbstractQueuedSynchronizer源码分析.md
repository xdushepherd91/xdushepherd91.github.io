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
release方法的调用逻辑还是比较简单的:
1. 调用子类的tryRelease方法，一般在ReentrantLock等类中实现。
   1. 如果该方法实现返回为true，即当前线程已经完全释放了了锁，进入if判断，调用unparkSuccessor来唤醒等待线程。该方法下一节有详细介绍。
   2. tryRelease方法返回为false，说明锁未完全释放，release方法返回false。




#### unparkSuccessor方法

对于该方法，需要和前面的parkAndCheckInterrupt方法对应起来，我们现在要做的事情就是将之前被park的线程unpark，使其重新进入线程的调度队列中。具体逻辑
1. 更新当前线程对应的node的waitStatus状态
2. 找到head后的第一个未被cancele的节点 
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

### condition相关

在使用可重入锁的时候，一直很好奇condition是怎么实现的，这其实也是在AQS中实现的，AQS中的ConditionObject实现了Condition。

#### 组件

在ConditionObject类中，有两个node类型的组件
1. firstWaiter
2. lastWaiter

那么不言自明，ConditionObject内部也是有一个队列的，即由上两个组件组成的等待队列


#### await方法



##### unlinkCancelledWaiters方法

该方法的目的是，从以firstWaiter为头，lastWaiter为尾的等待队列上，从firstWaiter开始，
对waitStatus不为CONDITION状态的节点，进行剔除动作。具体逻辑
1. 从头结点开始，将当前节点引用赋值给变量t，将next设置为t.nextWatier引用。
2. 如果t不为CONDITION状态，那么，我们就要考虑剔除t了。具体过程
   1. 如果trail还未设置，那么直接领firstWaiter=next
   2. 否则，trail设置了的话，将t.nextWaiter引用赋值给trail.next即可。
   3. 如果next为null，那么说明没有后续等待节点了，直接将trail引用赋值给lastWaiter即可，整个方法就要结束了。
3. 如果是的话，就把t的引用在给trail（trail是用来记录上一个有效node的引用的）。
4. 将next赋值给t，循环继续，知道赋值给t的next为null的时候结束

````java
    private void unlinkCancelledWaiters() {
        Node t = firstWaiter;
        Node trail = null;
        while (t != null) {
            Node next = t.nextWaiter;
            if (t.waitStatus != Node.CONDITION) {
                t.nextWaiter = null;
                if (trail == null)
                    firstWaiter = next;
                else
                    trail.nextWaiter = next;
                if (next == null)
                    lastWaiter = trail;
            }
            else
                trail = t;
            t = next;
        }
    }
````


##### addConditionWaiter方法

逻辑：
1. 找到lastWaiter，并向前查找到第一个未被取消的condition为止。
2. 新建一个waitStatus为Node.CONDITION,thread为当前线程的节点。
3. 如果此时的t为null，那么firstWaiter设置为当前节点
4. 否则，将lastWaiter的nextWaiter设置为当前节点
5. 将lastWaiter设置为当前node

注意：这里之所以在变量修改的时候不需要加锁，是因为我们当前在锁内部，因此天然安全。

````java
    private Node addConditionWaiter() {
        Node t = lastWaiter;
        // If lastWaiter is cancelled, clean out.
        if (t != null && t.waitStatus != Node.CONDITION) {
            unlinkCancelledWaiters();
            t = lastWaiter;
        }
        Node node = new Node(Thread.currentThread(), Node.CONDITION);
        if (t == null)
            firstWaiter = node;
        else
            t.nextWaiter = node;
        lastWaiter = node;
        return node;
    }
````
##### fullyRelease方法

方法的目的很简单，那就是释放锁，由于同一个线程可能多次获取锁，所以AQS的state是可能比1更大的，该方法的逻辑就是彻底的释放锁，
将AQS的state变为0，将锁释放

````java
    final int fullyRelease(Node node) {
        boolean failed = true;
        try {
            int savedState = getState();
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }

````

##### isOnSyncQueue方法

该方法检查入参node是否在队列中，逻辑如下：
1. 前两个if判断很容易理解，快速确定node在或者不在一个队列中
2. findNodeFromTail方法的逻辑是从AQS等待队列的尾部开始，向前查找，
如果查找到了一个节点正好等于当前节点，那么返回true，否则一直查找，
直到查找将队列遍历完毕还未找到的话，就返回false。

````java
    final boolean isOnSyncQueue(Node node) {
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        if (node.next != null) // If has successor, it must be on queue
            return true;
        return findNodeFromTail(node);
    }

    private boolean findNodeFromTail(Node node) {
        Node t = tail;
        for (;;) {
            if (t == node)
                return true;
            if (t == null)
                return false;
            t = t.prev;
        }
    }    
````

####### 小结

    isOnSyncQueue如果一直返回false，wait方法就会在一个循环中持续执行，并且有可能被park掉。
总之，如果该方法一直返回为true，说明当前线程一直在ConditonObject的等待队列里，而未能进入到
AQS的锁队列中去，需要一直等待。
    这就是condition的wait方法的核心逻辑所在

##### acquireQueued方法

该方法在前文中有介绍，再去复习一下即可。这里简单说一下其目的。

在当前节点重新进入到AQS的等待队列中去的时候，使用该方法重新获取到锁，并设置为之前的的重入次数。

##### 总结

结合前面小结的内容，我们可以很好理解await方法如何实现的了，后面再来看看signal方法吧。

````java
    public final void await() throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        Node node = addConditionWaiter();
        int savedState = fullyRelease(node);
        int interruptMode = 0;
        while (!isOnSyncQueue(node)) {
            LockSupport.park(this);
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
        }
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null) // clean up if cancelled
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
    }
````

#### signal方法

首先声明，在这些方法执行的时候，是不存在竞争的可能的，这是在锁内部执行的。

####@　doSignal方法

在doSignal内部，其执行逻辑是
1. 首先将当前节点first的nextWaiter节点引用记录到firstWaiter记录下来。
2. 判断记录的nextWaiter是否为null，是的话，说明到了队尾，需要把lastWaier设置为null了。
3. 当前节点的nextWaiter设置为null（gc）。
4. 如果transferForSignal方法执行返回为true，说明将当前节点加入到了AQS的等待队列。循环结束。
5. 如果返回为false，即node的waitStatus在设置的时候不符合预期（见下文）,我们要直接跳过该节点了。
6. 将当前节点记录为之前存下来的nextWaiter节点，如果其部位null，说明还有线程在等待，那么循环继续，否则
循环结束。

我们看看transferForSignal方法的逻辑吧：
1. 如果试图将当前node的waitStatus通过cas设置为0失败，那么就返回false，上述的循环继续。
2. 如果设置成功，那就需要将当前节点加入到AQS的锁队列中去，以为这当前node对应线程马上就可以
有资格获取到锁了。
3. 在enq入队后，会将node的前一个node返回来，如果

````java
    private void doSignal(Node first) {
        do {
            if ( (firstWaiter = first.nextWaiter) == null)
                lastWaiter = null;
            first.nextWaiter = null;
        } while (!transferForSignal(first) &&
                    (first = firstWaiter) != null);
    }

    final boolean transferForSignal(Node node) {

        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        Node p = enq(node);
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
````

#### 小节

通过上文，我们可以很容易理解signal函数的逻辑：将等待队列中的第一个节点加入到
AQS的等待队列中去。

````java
    public final void signal() {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        Node first = firstWaiter;
        if (first != null)
            doSignal(first);
    }
````


### 总结

本文大的分为两个部分：
1. acquire和release。对应了lock和unlock方法。
2. condition.await和condition.signal方法。

有一个遗留问题，就是waitStatus的状态转换全面梳理未进行，这个后续再来看看吧。

