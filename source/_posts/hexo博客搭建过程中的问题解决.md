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

这个部分需要解决以下两个问题：

1. 如何让更多人看到我的博客内容，而不是仅仅自己本地查看？

2. 我希望在公司，家里，甚至是任何一台新电脑上随时开始写博客并且部署，如何实现？


### 部署

[github pages](https://hexo.io/zh-cn/docs/github-pages)部署，参考该页面中的部署方式，我们会得到这样一个结果：

1. 第一部分中的博客目录完全备份到github仓库中。这样，我们就不需要担心博客内容丢失了。

2. 任何时候想要新写一个博客，只需要hexo new post post-name。写完之后将博客加入仓库，然后推送到github仓库。然后Travis自动将新的博客发布到博客分支之上。

3. 在任何一个安装了git，nodejs和hexo-cli的电脑上，我们只需要拉取仓库，然后就可以新建博客，开始写作了。

在第二部分一开始提出的问题，基本解决。

在实际操作过程中，有一些问题需要注意。

1. hexo官网教程指导创建的仓库，github pages只能解析master分支，而Travis自动化构建的时候，将博客内容发布到了gh-pages分支。这会导致博客内容无法浏览。解决方案如下：
   1. 在博客主目录下,git checkout -b blog-source；git push origin blog-source:blog-source;
   2. git checkout master;git rm .;rm -rf *;git add .;git commit -m "清空master分支";git pull origin master;git push origin master;
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

2. next主题也是一个git仓库，无法添加到博客仓库。解决方法如下：
   1. 进入themes/next目录，执行 rm -rf .git。
   2. 返回主目录，git add themes/next。

## 总结

经过如上步骤，我们得到了一个基本功能完善，由github仓库备份，随时可用的hexo博客系统。

记忆力太差，博客来补。

























