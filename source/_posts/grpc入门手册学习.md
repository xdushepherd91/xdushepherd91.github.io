---
title: grpc入门手册学习
tags:
  - go
  - rpc框架
categories:
  - docker
date: 2019-11-08 13:46:32
---


### 概述

本篇文章之所以位于docker目录，是因为是在学习docker底层原理的过程中，涉及到了grpc框架，需要对该框架有一个完整的了解。
本文基本上是对[grpc官方文档](https://grpc.io/docs/guides/)的一个总结归纳，没有新鲜的东西。

主要包括以下一些内容：
1. gRPC
2. buffers.gRPC协议
3. Interface Definition Language（IDL）
4. 消息交换格式

#### 简介

使用gRPC框架，你可以在一个客户端程序中直接调用位于另一个机器上的方法，就好像在调用一个本地方法一样，这使得我们可以很容易创建一个分布式的应用程序
和服务。

和其他的rpc系统一样，gRPC也是继续以下一些事实：

一、  
1. 定义一个服务
2. 声明可以被远程调用的方法的参数和返回类型

二、在服务端  
1. 实现我们已经定义好的接口
2. 启动一个gRPC服务来处理来自客户端的调用请求

三、在客户端  
1. 在客户端有一个存根（某些语言中称之为客户端）同样提供了这些方法

![title](/images/gRPC/1.svg)

### Protocol Buffers

默认情况下，gRPC使用Google开源用于序列化数据的第三方库。下面是一个对其工作机制的快速介绍。

第一步，自己新建一个扩展名为.proto的文本文件中定义一个你想要序列化的数据格式。示例如下：

``` go
message Person {
  string name = 1;
  int32 id = 2;
  bool has_ponycopter = 3;
}
```

第二步，在定义了数据结构之后，使用protocol buffer提供的编译工具protoc来生成指定语言的数据类，对于上面的定义，可以生成一个名为Person的类。

更进一步的，你可以在proto文件中定义服务，方法，入参，返参等等

``` go
// The greeter service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
````

上述文件也可以使用protoc来生成对应的文件。

#### 协议版本

protocol buffer是有多个版本的，不同版本的语法可能有不同。

### 参考

[入门指南](https://grpc.io/docs/guides/)
[核心概念](https://grpc.io/docs/guides/concepts/)
[gRPC示例go语言版本](https://grpc.io/docs/quickstart/go/)






