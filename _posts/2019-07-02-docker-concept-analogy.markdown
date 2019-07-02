---
layout: post
title:  "Docker几个概念的类比"
date: 2019-07-01 11:24:00
categories: development
tags: [Docker]
comments: true
---

Docker的官方网站上列出了Docker的几个核心概念：image, container,service,stack,swarm

为了便于理解将这几个概念和程序员熟悉的编程日常做一个类比：

1. image 就是类似于代码中的类class 它声明了这个class应有的属性和功能
2. container 就是class的实例化instance 或者叫 对象object
3. service 就是相当于模块module ，模块里面有好几个类(image) ,这个模块运行的时候自然也会生成好几个object（container），合起来提供一个服务
4. stack 就像是一个多模块app,当然也可以是单模块的
5. swarm 就是想这个app 可以多服务器部署

**这样类比不是很精确，只是为了建立一个直观的docker概念印象**