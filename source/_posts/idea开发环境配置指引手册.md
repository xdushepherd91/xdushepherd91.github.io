---
title: idea开发环境配置指引手册
tags:
  - idea
  - ide
  - 开发工具
categories:
  - 开发效率
date: 2019-12-05 14:31:40
---

### 概述

在平时的工作中，难免遇到重装系统，换电脑之类的问题，这些问题解决之后，就要重新搭建我们的开发环境，这就
包括idea的安装和配置问题了，本篇文章，旨在记录搭建idea开发环境的各个方面，激活，汉化，插件，类注释，方法注释等等。

<!-- more -->

### 激活

网上激活的文章一大堆，这里不再赘述，推荐的博客可能过了不久就失效了，激活现用现查就可以了。

### 汉化

我是在使用idea很久之后，才想到要对其进行汉化，中国人用汉化的界面还是更舒服一些的。下面是激活相关内容参考
[TranslatorX仓库](https://github.com/pingfangx/TranslatorX)，具体步骤很简单，参照readme的指导就可以了。

### 插件

关于插件的博客也是不胜枚举，本博客主要记录一下我测试过感觉对我来说很有用的一些插件

#### stackoverflow

在右键中添加了search stackoverflow按钮，当我们选中了想要在stackoverflow中查询的文本之后，右键鼠标，点击该按钮，
即可发起查询，这需要我们主机的网络可以四通八达。

总的来说，很方便。

#### TranslationPlugin

这个插件呢，可有可无吧，试了一下，如果我们不想下载有道词典的话，这个插件也可以满足日常开发使用

#### Alibaba Java Coding Guidelines

顾名思义，代码规范检查，很好的一个插件，具体使用方法见[阿里巴巴 Java 开发规约插件初体验](https://www.cnblogs.com/mafly/p/aliPlugin.html)

#### Lombok

必备神器了，极大的减少代码量并且美化代码。[Lombok介绍、使用方法和总结](https://www.cnblogs.com/heyonggang/p/8638374.html)

####　CamelCase

将不是驼峰格式的名称，快速转成驼峰格式，安装好后，选中要修改的名称，按快捷键shift+alt+u。

可有可无，习惯了还是比较有用的。　

#### Rainbow Brackets

安装了用一下就知道了，帮助我们快速确认代码块。

### 类注释和方法注释

关于这块的，参考[IntelliJ IDEA技巧-添加类注释和方法注释](https://yashuning.github.io/2018/04/28/IntelliJ-IDEA%E6%8A%80%E5%B7%A7-%E6%B7%BB%E5%8A%A0%E7%B1%BB%E6%B3%A8%E9%87%8A%E5%92%8C%E6%96%B9%E6%B3%95%E6%B3%A8%E9%87%8A/)
<<<<<<< HEAD
在原文的基础上，我根据配置过程，对方法注释做了一些修改:

#### 方法注释模板

````java
/**
* @description $todo$
$params$
* @return $return$
* @author xdushepherd91
* @date $date$ $time$
**/
````

#### params表达式

````groovy
groovyScript("def result=''; def params=\"${_1}\".replaceAll('[\\\\[|\\\\]|\\\\s]', '').split(',').toList(); for(i = 0; i < params.size(); i++) {result+='* @param ' + params[i] + ((i < params.size() - 1) ? ':\\r\\n' : '')}; return result", methodParameters())
````



=======
>>>>>>> 5c7db7a42a69578c34e100a403bd19bde493c0e5


### 配置导出

根据我们习惯配置好了一个idea的环境，推荐将配置导出，并存储再github仓库中，以备不时之需。

















