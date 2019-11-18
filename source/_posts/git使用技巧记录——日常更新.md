---
title: git使用技巧记录——日常更新
tags:
  - 开发工具
  - git使用技巧
categories:
  - git
date: 2019-11-18 13:48:12
---

### 介绍

git作为非常有用的开发工具，值得每个开发人员熟练掌握，并且，对于git的掌握越深入，对我们的开发工作就越有利。本片文章，
是我日常使用git时的一些记录，并且会时常更新，如果已经记录过的命令，有了遗忘，我会在该命令下记录再次学习的时间，并且学习次数+1。

### git diff

git diff命令是用来对比文件差异的，可以使工作目录与索引库，仓库以及各个提交版本的不同，主要使用方法：

1. git diff filename。 工作目录和索引库异同。
2. git diff --staged filename。 索引库和仓库的异同。
3. git diff HEAD filename。 工作目录和仓库的异同。

除此之外，我们也可以比较不同提交之间的差异

学习时间：
2019/11/18 13:55
复习次数：1


### 修改远程分支名称

1. git push --delete origin branch-name  删除远程分支

2. git branch -m branch-name new-branch-name 修改本地分支名称

3. git push origin new-branch-name 将新的分支名称推送到远程

由于输入失误，将分支名称写错了，同时本人略带强迫症，因此从网上找了相关教程，将远程分支名称做了修改。


学习时间：
2019/11/19 16:27
复习次数：1






