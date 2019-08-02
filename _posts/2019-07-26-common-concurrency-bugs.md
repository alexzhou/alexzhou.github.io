---
layout: post
title:  "常见的并发问题"
date: 2019-07-26 18:24:00
categories: OperateSystem
tags: [threads,concurrency,OSTEP笔记,common bugs]
comments: true
---
有研究者通过研究Mysql,Mozilla,Apache,OpenOffice中的关于并发的问题发现，非死锁问题大概是死锁问题数量的两倍  
####非死锁问题
只列举一些常见的问题  
##### 1. Atomicity Violation 违反原子性 
```c
Thread 1::
if (thd->proc_info) {
    fputs(thd->proc_info, ...);
}

Thread 2::
thd->proc_info = NULL;
```

比如当线程1检查完proc_info 的值之后打算往里面写操作的时候，系统中断了线程1，线程2操作proc_info被置NULL;这个过程就违法了原子性操作。  
解决方法就是加个pthread_mutex_t锁  

##### 2. Order-Violation Bugs 顺序冲突
```c
Thread 1::
void init() {
  mThread = PR_CreateThread(mMain, ...);
}
6
Thread 2::
void mMain(...) {
  mState = mThread->State;
}
```

这个就更好理解了，thread2 以为 thread1 已经创建好了线程mThread ，但是有可能没创建好，所以需要condition variable 约束一下  

#### 死锁问题

死锁发生的主要原因：  
1. Mutual exclusion
很难不用mutual exclusion, 或许可以在硬件上找到替代方案
2. Hold-and-wait
fix : 让某个线程一次性获得所有锁  
3. No preemption  
fix:设计优先级  
4. Circular wait   
fix : 让线程按顺序获取锁 Total Ordering 或者 Partial Ordering 






