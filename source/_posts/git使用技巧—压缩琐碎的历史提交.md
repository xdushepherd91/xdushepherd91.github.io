---
title: git使用技巧—压缩琐碎的历史提交
tags:
  - 开发工具
  - git使用技巧
categories:
  - git
date: 2019-11-27 09:37:54
---


### 概述

在开发runc仿制品[toy-container](https://github.com/xdushepherd91/toy-container)的时候，在调试过程中，经常会对代码做简单的修改，
然后做了提交，虽然有意识的使用git commit --amend来共享上一次提交的日志，但是，不免还是会有一些琐碎的提交出现，因此想要使用git提供的
命令行工具，将历史提交给压缩一下，让提交变得整洁一些。

<!-- more -->

### 步骤

git checkout -b reabse-branch  //新建一个分支，防止在rebase期间出现了错误操作，影响了整个代码库
git rebase -i start-point // rebase操作的起始提交，
edit rebase file // 上一步中的命令会自动打开一个文本编辑框，我们可以在其中指定我们如何压缩提交
solve conflict //*  在rebase过程中，可能出现冲突，需要手动解决
git add . //* 如果出现了冲突，我们在解决了之后，将其添加到index中
git rebase --continue / git rebase --abort // * 如果出现了冲突，解决完之后，执行continue就可以完成rebase操作，否则，执行abort放弃此次rebase操作

### 示例

#### 创建项目

首先是创建一个有多次提交的项目，如下:
![](/images/git-rebase/1.png)

#### 压缩提交

接下来，我想要将提交3和提交2合并为同一个提交，提交4和提交5合并为同一个提交，命令如下:
![](/images/git-rebase/2.png)

git rebase -i 77b91cc执行得到如下界面:
![](/images/git-rebase/4.png)

**注意:文件中并未包含77b91cc**,可以看到，pick开头的信息是按照时间序逐行书写的，并且下方注释部分有操作教程，我们的目的是
分别合并提交2和提交3，提交4和提交5，那么根据教程方法如下：
![](/images/git-rebase/5.png)

#### 重新编辑提交信息

保存该文件(使用vim的保存方式),会得到如下文件界面，该界面是用来编写压缩之后的提交日志的。
![](/images/git-rebase/6.png)

我们此次修改2和3压缩后的提交信息，不修改4和5压缩后的提交信息，做个对比。
![](/images/git-rebase/7.png)

#### 查看日志

结束了上述一通操作，我们看看现在的历史提交信息是否变成了我们想要的样子？答案是肯定的。见下图
![](/images/git-rebase/8.png)

### 总结

本文仅仅是一个简单的git rebase -i进行提交日志合并的介绍和示例，并未包含以下一些情况：
1. 分支合并解决冲突之后的提交日志合并
2. rebase过程中出现冲突解决
3. 其他


### 参考

[(Git)合并多个commit](https://segmentfault.com/a/1190000007748862)






























