---
title: docker底层原理————overlayFS介绍与实践
tags:
  - linux
  - filesystem
  - docker
categories:
  - filesystem
date: 2019-11-05 14:29:17
---


### 概述

在docker底层原理的学习过程中，接触到了UnionFS的概念。

UnionFS，顾名思义，是联合文件系统的意思。那怎么个联合法呢？举个简单的例子：
假设，我们有两个文件夹A和文件夹B，A文件夹下有文件a1,a2,a3,B文件夹下有文件b1，b2，b3。
我们通过创建一个文件夹C，并且使用UnionFS，使得我们可以再C文件夹下可以直接访问a1,a2,a3,b1,b2,b3。
这样我们就把A文件夹和B文件夹联合起来了。

当然，在实际的实现过程中，概念会更多。

UnionFS的实现是比较多的，目前docker中使用的UnionFS的实现是overlay2，而我在网上找到的相关资料并不多，
处于好奇，专门对其进行了一些实际操作，并记录下来。

### 简介

overlayFS允许一个可读写的目录覆盖在多个只读的目录之上，这些目录就构成了多个层(layer)，而当我们在联合起来的目录中工作的时候，
我们所有的写入操作最终都进入了最上层的读写层目录中，这一点我后续会通过命令展示。

overlayFS与其他的unionFS的实现最终要的不同在于，当一个文件打开之后，所有的操作都直接和底层的文件系统打交道，这样可以保证操作的
性能。

在Linux kernel 3.18之后，overlayFS已经集成到内核中去了。我们在该版本之后的Linux内核中，不需要安装，就可以开始使用overlayFS了。


### 使用

#### 可读写的联合文件系统

我们使用如下命令来挂在一个overlay文件系统

mount -t overlay overlay -o lowerdir=/lower,upperdir=/upper,workdir=/work /merged

其中，
1. lowerdir的值可以是一些的文件夹列表，使用:分开,这些事只读层。
2. merged文件夹是最终联合起来的文件系统，我们可以在merged文件夹中访问所有lowerdir和upperdir中的内容
3. 在merged文件夹所做的所有修改，最终都会存储到upperdir目录中
4. workdir指定的目录需要和upperdir位于同一目录中
5. 文件的覆盖顺序，upperdir目录拥有最高覆盖权限，lowerdir按照mount时从左到右的顺序，权重依次降低，左边的覆盖右边的同名文件或者文件夹。
注意，覆盖仅仅在mount时按照此顺序，一旦mount成功后，按照文件出现的早晚覆盖，出现早的会屏蔽出现晚的同名文件。除非修改是在merged目录中进行。

#### 开机自动挂载

在/etc/fstab中实现自动挂载的格式如下，本人暂未实践

overlay /merged overlay noauto,x-systemd.automount,lowerdir=/lower,upperdir=/upper,workdir=/work 0 0

#### 只读联合文件系统

mount -t overlay overlay -o lowerdir=/lower1:/lower2 /merged


### overlayFS相关演示


![](/images/overlay/1.png)
![](/images/overlay/2.png)
![](/images/overlay/3.png)
![](/images/overlay/4.png)
![](/images/overlay/5.png)


### 总结

overlayFS可以在多种应用场合中使用，我们这里关注在docker中的使用方式。
在docker中个，多个镜像可以看做是多个lowerdir，只读层，当我们启动一个容器的时候，使用overlayFS将多个镜像
联合挂载到一个目录，并使得容器中可以对目录中的文件进行读写操作，但读写操作仅仅会影响到当前容器的upperdir和
当前目录，并不会影响到镜像中的数据。

### 参考链接

[Docker技术三大要点：cgroup, namespace和unionFS的理解](https://www.jianshu.com/p/47c4a06a84a4)
[Overlay filesystem](https://wiki.archlinux.org/index.php/Overlay_filesystem)
[Docker技术原理之Linux UnionFS（容器镜像）](https://blog.csdn.net/songcf_faith/article/details/82787946)

