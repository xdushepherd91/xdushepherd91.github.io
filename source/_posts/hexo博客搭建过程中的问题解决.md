---
title: hexo博客搭建过程中的问题解决
tags:
  - hexo
  - github pages
  - 自动化部署
categories:
  - hexo博客搭建
date: 2019-10-10 16:49:30
---

### 概述

本文分为几个部分，其中大部分都是官网的内容，这里只提供框架，方便日后对照。
1. hexo博客的本地搭建。比较简单，基本上按照官网的流程走即可。
2. hexo博客部署问题解决

### 本地环境搭建

在使用hexo搭建博客的过程中，首先要面临安装的问题。在一个干净的环境中，我们需要以此安装：
1. nodejs
2. git
3. hexo-cli
具体参考官方文档[hexo环境搭建](https://hexo.io/zh-cn/docs/)

### 第一篇博文

``` bash
## 生成博客，查看博客的基本结构，具体每个目录的介绍看官网。(https://hexo.io/zh-cn/docs/setup)
hexo init blog-name
cd blog-name
npm install
ls
## _config.yml文件的配置博客，参考官网 (https://hexo.io/zh-cn/docs/configuration)

```
除了生成博客文章，hexo new命令还可以生成其他的












