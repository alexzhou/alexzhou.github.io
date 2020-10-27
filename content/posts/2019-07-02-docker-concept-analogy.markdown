---
layout: post
title:  "Docker几个概念的类比"
date: 2019-07-02 17:37:00
categories: development
tags: [Docker]
comments: true
---

Docker的官方网站上列出了Docker的几个核心概念：image, container,service,stack,swarm

为了便于理解将这几个概念和程序员熟悉的编程日常做一个类比：

1. image 就是类似于代码中的类class 它声明了这个class应有的属性和功能
2. container 就是class的实例化instance 或者叫 对象object
3. service 就像是class的调用，怎么初始化，以及生成几个实例
4. stack 就像一个app,它实际就是一组service
5. swarm 就是可以为app提供分布式部署的工具,可以把app部署到多个服务器上

**这样类比不是很精确，只是为了建立一个直观的docker概念印象**