---
title: docker1.9源代码结构及相关资源链接分享
tags:
  - linux
  - filesystem
  - docker
categories:
  - docker
  - linux内核
date: 2019-11-06 16:41:29
---

### 概述

去年八月份左右，曾经做了一段时间的k8s部署相关的工作，因此对docker具有一定的使用基础并大致了解其相关概念。
最近一周开始看docker源码，这里对其做一些总结和资料汇总。在docker1.9这个版本里面，关于容器启动和运行的核心代码
已经被抽离到了containerd和runC之中了。当然，这是在我开始阅读docker源代码的时候，所不知道的，关于containerd和runC的相关
介绍，见文末。

[从零手动创建一个容器](https://ericchiang.github.io/post/containers-from-scratch/)



### 源码结构

go语言相关的项目结构，相对是比较自由的。从github上clone下来的docker-ce仓库中，主要有一个components子目录，该子目录下有两个关键的子目录：
1. cli子目录。这个目录是docker client相关的代码。
2. engine子目录。这个目录是docker engine相关的代码。
以上两个子目录中都有如下路径的go文件cmd/docker/docker.go，docker.go文件中包含了client和engin的主函数，我们进行代码分析的入口所在。

### 架构及相关第三方类库介绍

首先，我们必须明确，docker是一个c/s架构的开源项目，对于  docker run -it --name newos docker.io/centos 这样一条命令，实际的执行流程如下：
1. docker client 从主函数开始，解析函数命令及相关参数。
2. docker client 根据命令及入参构造请求地址及请求参数并向docker engine中的指定服务地址发出请求
3. docker engine 根据入参地址调用对应的处理方法。
4. docker engine 的处理方法根据入参是用containerd的客户端引用创建容器，创建容器任务，启动容器等等

对于docker client，主要涉及了如下两个第三方类库，用于命令行参数解析和运行，需要自行去运行官方demo即可。
[cobra](https://github.com/spf13/cobra)，该类库用于创建POSIX兼容的命令行工具。
[pflag](https://github.com/spf13/pflag)，该类库用于解析命令行参数。
基本上docker client就是如此简单，解析命令行参数构建请求url，使用官方类库发出http请求等等。

对于docker engine，主要涉及了包括cobra，pflag在内的多个类库，具体如下：
[mux](https://github.com/gorilla/mux),该类库用于请求路由工作。
[containerd](https://containerd.io/docs/getting-started/),由docker官方开源的类库。
[runC](https://github.com/opencontainers/runc),有docker官方开源的容器运行时类库。

### docker engine源码关键点总结

#### 启动过程要点

docker engine的启动过程包括如下一些关键点：

1. 根据命令启动，使用到cobra相关知识。
2. 需要启动http监听接口，并使用mux进行请求路由工作。
3. 需要启动containerd服务及客户端。

其他，docker1.9中的源代码中关于配置的结构体比较多，不知道是重构的遗留还是官方刻意为之，总之看起来比较绕，
不过耐心的总结，还是可以形成概念的。

#### 请求处理

docker engine的主要工作就是处理请求，和其他的web服务器一样，根据请求地址路由到
相应的处理方法，如果为容器相关操作，那么就需要涉及到containerd相关调用，这又需要对containerd有一定的了解。

### 容器核心概念资源分享

先来点题外话，我在docker源码阅读过程中，有如下收获：
1. 对刚刚学习的go语言有了更进一步地认识。
2. 对go语言的类库有了新的认识。
3. 对linux底层有了更进一步认识。

我们都知道docker的关键技术有namespace，cgroup，unionFS等，但是很多又是泛泛而谈的，talk is cheap，show me the code！
下面列举了一些我在研究docker源码过程中比较有收获的一些博客：

1. namespace。[Docker基础技术-Linux Namespace](https://www.jianshu.com/p/353eb8d8eb05),该博客对namespace做了详细的介绍，并且
对每一种namespace都提供了c语言代码来进行实践，我们只需要按照代码一步步实践，就可以对docker源码对namespace的应用有一个切实的了解。
[代码地址](https://github.com/shishujuan/docker-basis)。如果不幸，链接不存在，可以在我的备份中找到：
文章备份可以联系我，备份于onenote。
[代码备份](https://github.com/xdushepherd91/docker-basis)

2. unionFS。关于unionFS的内容，在我的博客[docker底层原理————overlayFS介绍与实践](https://xdushepherd91.github.io/2019/11/05/overlayFS%E4%BB%8B%E7%BB%8D%E4%B8%8E%E5%AE%9E%E8%B7%B5/)，中有一定的介绍，相信大家看了之后，会有一定的收获。

3. cgroup。[Docker之Linux Cgroup实践](https://www.jianshu.com/p/b02bf3b3f265)。Linux Cgroup最主要的作用是为一个进程组设置资源使用的上限，这些资源包括CPU、内存、磁盘、网络等。在linux中，Cgroup给用户提供的操作接口是文件系统，其以文件和目录的方式组织在/sys/fs/cgroup路径下。  
[linux内核cgroups介绍](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)  
[Linux资源管理之cgroups简介](https://tech.meituan.com/2015/03/31/cgroups.html)
[左耳朵耗子](https://coolshell.cn/articles/17049.html)



### 总结

很遗憾，在docker-ce的源码中，没有看到底层cgroup，namespace，unionFS相关的代码，这些更底层的代码已经被移动到了containerd代码库中，
不过，在阅读docker-ce源码的过程中，通过实际操作网络上的相关资源，我对docker底层实现有了更深入的了解，后续，我会对containerd项目的
源代码进行阅读分析和分享


福利，一个书写有些混乱，但是干货满满的博客:[Docker底层技术](https://www.jianshu.com/p/ef41503b8f87)


















































