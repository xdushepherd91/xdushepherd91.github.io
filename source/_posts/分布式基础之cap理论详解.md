---
title: 分布式基础之cap理论详解
tags:
  - null
categories:
  - null
date: 2019-12-02 19:28:58
---

### 概述

在刚开始接触分布式就听说过cap理论，大致看过一点，其实不是很明白，也是由于当时自身水平受限。最近开始找工作，遇到了好几个
面试官询问cap相关知识，当时只能大致说一下cap的基本内容：
1. 一致性，可用性，分区容错性
2. 三者只能达其二

仅此而已，非常浅显，这是不够的，因此，特地各种查找资料，寻找示例，试图理解cap理论。

### 关于p

p，即Partition tolerance，很多翻译都说是分区容错性，在我理解了一些之后，我觉得这个翻译是有问题的，翻译成分区容忍更好。
什么意思，就是容忍系统有网络分区，处于不同网络的子系统之间可能因为网络不可达或者其他原因造成的系统被分为多个网络区。

p对我来说最难理解，不过当翻译变过来之后，就感觉很容易理解了。

一般来说，在分布式系统中，p几乎是必然存在的，为什么？因为网络层面的问题不可避免。

那么我们要如何保证分区容忍性呢？

1. 当你一个数据项只在一个节点中保存，那么分区出现后，和这个节点不连通的部分就访问不到这个数据了。这时分区就是无法容忍的。
2. 提高分区容忍性的办法就是一个数据项复制到多个节点上，那么出现分区之后，这一数据项就可能分布到各个区里。容忍性就提高了。

保证了分区容忍性，就带来了另一个问题，数据的一致性。

为了保证数据一致性，数据同步的时间就越长，可用性就会降低。


<!-- more -->


### 关于a和c

a，即Availability，可用性。c，即Consistency，一致性，一般指的是各个分系统中数据是否一致。

在我们p是必须的同时，c和a就只能实现其中之一了，这两者结合来看会更好一些。

1. 保证一致性。如果要保证一致性，那么它要把所有的消息复制到所有的节点，但是我们的分布式系统是p的，也就是子系统之间可能是
不可以互相访问的，那么如果出现了分区，信息是必然无法复制，为了保证一致性，我们必须给客户端返回处理失败的结果，那这就意味着
此刻系统是不可用的了，那就丧失了A。

2. 保证可用性。如果要保证可用性，在系统出现分区之后，我们的消息只要到了系统的某个分区，即认为发送成功，由于分区的存在，整个
系统的一致性是必然无法保证的，于是就丧失了c。

3. 保证不分区。即不容忍分区，假设系统永远不出现网络分区？？？电信网络问题，主机宕机等等，这些是不可避免的，因此一般都要容忍分区的。


### 一致性再谈

为了保证可用性，我们一般来说，会对一致性进行一个折中，对一致性有多种实现：


1. 强一致性。强一致性很好理解，在一个更新操作完成之后，任何多个进程或者线程的访问都会返回最新的值。这是对用户最友好的，根据cap理论，
这种强一致性必然导致了可用性的降低。
2. 弱一致性。对比强一致性，弱一致性的意思就是不保证一个更新操作之后，其他的进程一定返回最新的值。
3. 最终一致性。弱一致性的特定形式。系统保证在没有后续更新的前提下，系统最终返回最新的更新值，在没有发生故障的前提下，不一致窗口的时间主要
取决通信延迟，系统负载和副本个数等的影响。
4. 顺序一致性。来自任意特定客户端的更新都会按其发送顺序被提交保持一致。


### 参考

[CAP理论中的P到底是个什么意思？ - Irons Du的回答 - 知乎](https://www.zhihu.com/question/54105974/answer/138030946)  
[CAP理论中的P到底是个什么意思？ - 邬江的回答 - 知乎](https://www.zhihu.com/question/54105974/answer/139037688)


