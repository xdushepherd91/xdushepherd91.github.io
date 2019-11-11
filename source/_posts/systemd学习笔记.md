---
title: systemd学习笔记
tags:
  - systemd
  - linux
categories:
  - linux
date: 2019-11-11 10:07:07
---

### 概述

在各种新技术的教程中，看到了服务启动从service apache2 start变到了systemctl start apache2.service，心中充满了疑惑，这是什么东西？
通过浏览各种博客，我下了一个初步的结论，service service-name start我不需要做过多的了解了，但是，systemctl start service-name.service，
作为未来一段时间的主流，我需要有一个透彻的理解。本文，也是我在学习systemd过程中，各种资源的整理，和自我理解的归纳。

### 参考

[Systemd 入门教程：命令篇](http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html)
[Systemd 入门教程：实战篇](http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-part-two.html)

以上为阮一峰写的systemd的相关教程，我个人认为写的非常好，基本上解答了我需要的systemd的基础知识。在此基础上，在网上随意找到一篇自定义systemd
的教程，基本上就理解了systemd的基本内容了。

### 关键点

大概梳理一下systemd服务的几个关键点：
1. /lib/systemd/system目录。一般来说，该目录下包含了系统中所有的可用的systemd风格的服务配置文件。也就是说，在该文件夹下有了一个有效的
service-name.service文件，我们就可以使用systemctl start service-name.service等命令来操控相关的服务了。
2. /etc/systemd/system目录。一般来说，该目录下配置了可以开机启动的服务，也就是1中服务文件的软连接。比如1中的service-name.servie服务文件，
我们使用systemctl enable service-name.service就可以在/etc/systemd/system文件夹下创建对应的软连接，进而可以完成开机启动
3. 服务配置文件的格式。一般来说，service配置文件都是以.service结尾的，内容分为三部分[Unit]小结，[Service]小结和[Install]小结。
   1. Unit小结主要是声明该服务的依赖，被依赖及其他与外部服务的交互关系内容
   2. Service小结主要是服务本身的相关配置内容，如何启动，如何停止等等
   3. Install小结，如何安装，这涉及到target类型的unit
4. target是什么? target是和service一样，是systemd中unit的一种，不过target是用来将一组unit组织到一起统一管理的。 
   1. target定义了一个服务组，什么意思？就是定义了一组同时启动的服务整体。
   2. test-target.target为例，假设我们在3中service-name.service的Install小结中定义了WantedBy=test-target.target，那么我们的使用systemctl enable service-name.service之后，会发现systemd会在/etc/systemd/systemd/test-target.target.wants目录下建立service-name.service的软连接，这和第2点中也相互呼应。
   3. 调用systemct start test-target.target，systemd 就会自动的启动service-name服务
5. 其他unit。除了service，target之外，还有其他的unit，比如device，mount,auotmount，timer等等，种类很多，但是基本上和service类似，如果需要使用，则可以
做针对性学习

















