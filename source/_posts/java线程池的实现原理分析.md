---
title: java线程池的实现原理分析
tags:
  - jdk
categories:
  - java
date: 2019-11-28 09:29:39
---

### 概述

昨晚在电话面试的时候，面试官问我Java多线程的实现方式，使用什么数据结构？说实话，我没有get到点，不过这也勾起了
我的疑问，Java的线程池到底是怎么实现的？

今天来就把jdk1.8的源码扯出来，准备看一下线程池这块的实现方式。线程池这块的核心类是ThreadPoolExecutor，也是本文分析
的主体。

可能网上的源码解析已经很多了，但是，我不自己看一下，还是觉得没理解，看了就得记一下。

早上被拉去开会了，中午刚好看了一会儿线程池源码，得，中午面试又问到这了，嘿嘿嘿。

<!-- more -->
### 组件介绍

#### ctl变量

ctl变量是ThreadPoolExecutor类的第一个变量，该变量是一个原子变量，其中存储了线程池的两个信息：
1. workerCount，表示有效的线程数目
2. runState, 表示当前线程池的运行状态，running，shutdown，stop或者其他。

代码如下：
````java
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    // Packing and unpacking ctl
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }
````

下面用一幅图来帮助大家理解

![](/images/jdk/threadpool/ctl.png)

从上图中，结合代码，我们很容易理解CAPACITY的用途，通过&操作从ctl变量中解析出workerCount和runState变量。对于其他几个变量入RUNNING,SHUTDOWN,STOP,TIDYING和TERMINATED，也就不难理解了。需要注意的是，RUNNING的数值最小。

代码中接下来有若干个私有方法，都是与ctl变量解析和操作相关的，无非两类：
1. workerCount的数目原子性增减
2. runState的原子性修改

#### 若干简单变量

1. workQueue变量。这个变量理解起来很简单，任务队列
2. mainLock变量。用来进行并发控制的，后续用到了很容易理解
3. largestPoolSize变量。用来存储线程池数目的历史峰值，比较好理解
4. completedTaskCount变量。存储线程池已经完成的任务总数
5. threadFactory变量。用来创建县城的工厂类
6. handler变量。线程池的拒绝策略
7. keepAliveTime变量。指定worker允许空闲的最大时间
8. allowCoreThreadTimeOut变量。是否允许核心线程被终止
9. corePoolSize。核心线程数
10. maximumPoolSize。最大线程数
其他。

#### Worker类

Worker类是线程池内部类，继承了AQS类并实现了Runnable接口。其内部变量如下：
1. Runnable实例。
2. Thread实例。该实例持有Worker本身。
3. completedTasks

对于一个Worker实例，在实际运行中，是由其持有的Thread实例通过start()方法，运行Worker实例的run方法，最终调用了runWorker方法，在runWorker方法中，调用了Runnable实例的run方法。

暂时可以这样简单的理解，每一个worker都是一个线程。

### runWorker方法及其他

#### runWorker

在上一节中，我们知道每一个worker实例都是一个工作的线程，那么我们想知道线程池怎么实现一直运行，怎么实现没有任务的时候将线程降低到核心线程数的？这些在runWorker方法中，我们都可以找到答案。

可以看到，runWorker方法内部有一个循环，这个循环如果结束，那么worker就执行结束了，线程就退出了，那么，线程池保留核心线程数量的原理肯定存在于while条件中的getTask()方法，这个方法，下一节会讲到。

我们先看一下while循环内部的逻辑吧。
1. 首先，获取任务。该循环通过getTask方法获取Runnable实例，实际上是从workQueue中获取任务。
2. beforeExecute和afterExecute钩子函数。这两个函数可以在扩展ThreadPoolExecutor的时候，进行扩展，默认情况先是不做什么事情的。
3. 可以看到task.run()调用，是核心逻辑，也就是在这里，执行了我们提交给线程池的任务。


````java
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
````

#### getTask

本节分析getTask方法的逻辑。可以看到，该方法内部是一个循环，其逻辑如下：
1. 首先是判断当前线程池的状态，如果已经属于shutdown状态了，自然需要进行一些处理，然后返回null，结合上一节，我们知道，每一个执行到这里的worker线程，都将会结束，迎来其线程的死亡。
2. 获取当前的worker数量，如果数量大于maximumPoolSize且任务队列是空，那么就对workerCount进行原子性减一，并返回null，结束当前worker线程。如果原子性减一失败，那么就进入下一次循环。这一点很好理解。
3. 从workQueue总take任务，获取到的任务不是null，就返回任务。如果是null，就要考虑接下的事情了。此时timedOut被设置为true，表示我们在任务队列中没有得到任务，要考虑是否降低线程数目了。
4. 可以看到timed 在当前workerCount大于核心线程数或者allowCoreThreadTimeOut设置为true时成立，从3中我们知道，timedOut在未能从workQueue中获取到数据。那么后续进入条件判断内部，返回null，导致worker线程结束
5. 如果allowCoreThreadTimeOut设置为ture，那么空闲期线程池也可以不存在核心线程在运行

````java
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
````

#### 小结

其实，到这里，我们已经理解了：
1. 线程池如何从阻塞队列中获取数据
2. 线程池如何维持一定数量的核心线程一直保持运行

接下来，我们看看execute方法。

### execute方法

有了上面的基础，我们再来看看execute方法吧，其执行逻辑有四个分支
1. 如果当前的workerCount小于corePoolSize，直接addWorker即可
2. 如果workerCount已经大于corePoolSize，就向workQueue中添加任务
3. 如果添加失败，那么调用addWorker方法，准备增加线程数
4. 如果以上都失败，说明无法加入任务了，调用reject方法，拒绝任务提交

可以看到核心在于addWorker方法，下一节就来看看addWorker方法吧

````java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }

````
#### addWorker方法

addWorker方法除了任务参数，还有一个core布尔量，表示着是否是在增加核心线程，这肯定是有区别的。

可以看到在retry循环，主要目的：
1. 如果线程池状态异常，那么直接返回false
2. 如果线程池数量大于maximumPoolSize，或者core等于true时当前线程数大于corePoolSize，那么直接返回false
3. 增加线程数统计，成功，循环结束，执行接下来的代码，否则，继续循环

接下来的代码，结合前面我们的代码分析，很好理解了：

1. 新创建一个worker，入参为传入的任务
2. 后去新建worker的线程实例
3. 加mainLock锁，因为我们需要更新workers和largestPoolSize这样的共享变量
4. 执行worker的线程实例的start方法，线程开始执行

其中还有一些流程控制的东西，很容易理解


````java
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
````


### 总结


这是我在今天阅读ThreadPoolExecutor源码时候的一些思路，当然，这是在总结之后的，开始的时候，也是无头苍蝇，乱撞。

希望这篇文章对想要知道ThreadPoolExecutor底层实现的朋友带来一点启发。

下面我需要对AQS也做一次总结，之前看过，不总结，忘了，好烦。