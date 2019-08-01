---
layout: post
title:  "基于事件的并发"
date: 2019-08-01 18:24:00
categories: OperateSystem
tags: [event,concurrency,OSTEP笔记,select,poll]
comments: trues
--- 

#### 事件循环的基础

之前一直用thread 来写并发的程序，但是在一些其他类型的并发程序也是用基于事件的方式，比如GUI程序以及http server 类的程序。  
基于事件的并发主要解决两个问题：  
1. anaging concurrency correctly in multi-threaded applications can be challenging  
2. in a multi-threaded application, the developer has little or no control over what is schedule at a given moment in time  

Event-based Concurrency 的简单定义： 
> you simply wait for something (i.e., an “event”) to occur; when it does, you check what type of
  12 EVENT-BASED CONCURRENCY (ADVANCED)
  event it is and do the small amount of work it requires (which may include issuing I/O requests, or scheduling other events for future handling, etc.). That’s it  
  
经典的事件程序：  
```c
//伪代码
while (1) {
    events = getEvents();
    for (e in events)
        processEvent(e); // event handler
}
```


#### 事件循环的API  
select()  OR  poll()  
系统提供的API可以做什么： 
> how exactly does an event-based server determine which events are taking place, in particular with regards to network and disk I/O? Specifically, how can an
event server tell if a message has arrived for it?

select() 需要注意的： 
1. note that it lets you check whether descriptors can be read from as well as written to  
2. note the timeout argument   

事件循环的一个特征： 
With an event-based approach, however, there are no other threads to
run: just the main event loop. And this implies that if an event handler
issues a call that blocks, the entire server will do just that: block until the
call completes.   

这个特征导出了另个一要求  no blocking calls are allowed.  
解决的这个要求的一些方法  
##### 1. Asynchronous I/O  

很多现代操作系统都会介绍asynchronous I/O这个新式的硬盘IO处理方式  

AIO control block （*nix） 


##### 2. 状态管理

UNIX SIGNALS  

#### 仍然存在的问题  
1. when systems moved from a single CPU to multiple CPUs, some of the simplicity of the event-based approach disappeared  
2. it does not integrate well with certain kinds of systems activity, such as paging  
3. event-based code can be hard to manage over time, as the exact semantics of various routines changes  
比如代码重构  从not block 重构成  block 的程序  
4. though asynchronous disk I/O is now possible on most platforms, it has taken a long time to get there , and it never quite integrates with asynchronous network I/O in as simple and uniform a manner as you might think  








