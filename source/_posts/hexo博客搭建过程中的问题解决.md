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

## 概述

本文分为几个部分，其中大部分都是官网的内容，这里只提供框架，方便日后对照。
1. hexo博客的本地搭建。比较简单，基本上按照官网的流程走即可。
2. hexo博客部署问题解决。

## 第一部分 博客搭建

### 本地环境搭建

在使用hexo搭建博客的过程中，首先要面临安装的问题。在一个干净的环境中，我们需要以此安装：
1. nodejs
2. git
3. hexo-cli
具体参考官方文档[hexo环境搭建](https://hexo.io/zh-cn/docs/)

### 第一篇博文

``` bash
## 1. 生成博客，查看博客的基本结构，具体每个目录的介绍看官网。(https://hexo.io/zh-cn/docs/setup)

hexo init blog-name
cd blog-name
npm install
ls

## 2. 生成第一篇博客

hexo new post "我的第一篇博客"

## 3. 生成静态文件

hexo generate

## 4. 启动本地博客服务。默认情况下，访问网址为： http://localhost:4000

hexo server

```

[指令](https://hexo.io/zh-cn/docs/commands)页面包含了更多详细的指令介绍。但是，我们最终的目的是在日常
写作中仅仅使用hexo new post title来生成博客，使用git相关命令推送博客内容即可。所以，对于其他的命令，知道就好。

### 主题和配置

在完成了基本的博客搭建工作之后，我们可以选择一款合适的主题，美化一下博客页面。我选择的是[next](http://theme-next.iissnan.com/)，非常经典的一款hexo
博客主题，基本上按照主题的指导，就可以配置出一款自己喜欢的博客皮肤了，当然，你也可以选择[其他主题](https://hexo.io/themes/)。


## 第二部分 博客部署

这个部分需要解决以下一些问题：

1. 将博客内容发布到互联网中，可以被更多人看到。

2. 我希望在公司，家里，甚至是任何一台新电脑上随时开始写博客。

3. 在公司，家里，或者任何环境中写到一半地博客可以简单地保存进度，然后在另一个环境中快速地拿到最新地进度，继续写作。

4. 如果我换了电脑了，我需要快速地将我的博客系统完整快速地复制一份出来


以上问题地答案就是，github，将整个博客（博客加主题配置等等）备份到github仓库中，并且：
1. 将完整地博客系统备份到仓库地其中一个分支，这个分支是我们日常工作地分支。
2. 将仓库的master分支作为github page目录，使用github提供的免费ci工具，实时从工作分支构建最新的博客并发布到
master分支上


### 部署

#### 基本的部署
[github pages](https://hexo.io/zh-cn/docs/github-pages)部署，参考该页面中的部署方式，我们会得到这样一个结果：

1. 第一部分中的博客目录完全备份到github仓库中。这样，我们就不需要担心博客内容丢失了。

2. 任何时候想要新写一个博客，只需要hexo new post post-name。写完之后将博客加入仓库，然后推送到github仓库。然后Travis自动将新的博客发布到博客分支之上。

3. 在任何一个安装了git，nodejs和hexo-cli的电脑上，我们只需要拉取仓库，然后就可以新建博客，开始写作了。

#### 问题解决

##### 博客访问问题

hexo官网教程指导创建的仓库，github pages只能解析master分支，而Travis自动化构建的时候，将博客内容发布到了gh-pages分支。这会导致博客内容无法浏览。

解决方案如下：
1. 创建日常写博客的分支,这里取名为blog-source后面的Travis配置文件中需要用到:
````bash
    git checkout -b blog-source;
    git push origin blog-source:blog-source;
````
2. 清空master分支内容，并与远程仓库进行同步
````bash
    git checkout master;
    git rm .;
    rm -rf *;
    git add .;
    git commit -m "清空master分支";
    git pull origin master;
    git push origin master;
````
3. 修改Travis配置文件.travis.yml
``` yml
    sudo: false
    language: node_js
    node_js:
      - 10 # use nodejs v10 LTS
    # cache: npm
    branches:
      only:
        - blog-source # 修改构建分支
    script:
      # - git clone https://github.com/iissnan/hexo-theme-next themes/next
      - hexo generate # generate static files
    deploy:
      provider: pages
      skip-cleanup: true
      github-token: $GH_TOKEN
      target_branch: master  ##修改目标分支为master
      keep-history: true
      on:
        branch: blog-source  ## 修改构建分支
      local-dir: public
```
##### 主题配置问题
next主题也是一个git仓库，无法添加到博客仓库。解决方法如下：
   1. cd themes/next;rm -rf .git;
   2. git add themes/next;
   3. git commit -m "备份主题配置";
   4. git pull;
   5. git push;

## 总结

经过如上步骤，我们得到了一个基本功能完善，由github仓库备份，随时可用的hexo博客系统。

记忆力太差，博客来补。

























