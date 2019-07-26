---
layout: post
title:  "Condition Variable 条件变量"
date: 2019-07-26 14:24:00
categories: OperateSystem
tags: [threads,concurrency,OSTEP笔记,condition variable,条件变量]
comments: true
---
### Condition Variables 的主要概念
A condition variable is an explicit queue that threads can put themselves on when some state of execution is not as desired;
some other thread, when it changes said state, can then wake one (or
more) of those waiting threads and thus allow them to continue (by signaling on the condition).
Condition Variable 的两个主要操作
1. pthread_cond_wait() 这个操作会自动释放mutex锁，如果cv条件不满足就block自己
2. pthread_cond_signal() 唤醒等待某个cv的线程
### Condition Variables的应用
#### 场景一：Parent Waiting For Child 父线程等待子线程  
最容一想到的办法使用一个共享变量shard variable
```c
volatile int done = 0;
void *child(void *arg) {
    printf("child\n");
    done = 1;
    return NULL;
}

int main(int argc, char *argv[]) {
    printf("parent: begin\n");
    pthread_t c;
    Pthread_create(&c, NULL, child, NULL); // create child
    while (done == 0); // spin
    printf("parent: end\n");
    return 0;
}
```
但是这种方式会导致父进程spin 而且浪费CPU资源，这种情况下就可以使用condition variable
```c
//Pthread 大写开头的是书中代码有对pthread原生方法的封装 不影响代码逻辑阅读
int done = 0;
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t c = PTHREAD_COND_INITIALIZER;
void thr_exit() {
    Pthread_mutex_lock(&m);
    done = 1;
    Pthread_cond_signal(&c);
    Pthread_mutex_unlock(&m);
}

void *child(void *arg) {
    printf("child\n");
    thr_exit();
    return NULL;
}

void thr_join() {
    Pthread_mutex_lock(&m);
    while (done == 0)
    	Pthread_cond_wait(&c, &m);
    Pthread_mutex_unlock(&m);
}

int main(int argc, char *argv[]) {
    printf("parent: begin\n");
    pthread_t p;
    Pthread_create(&p, NULL, child, NULL);
    thr_join();
    printf("parent: end\n");
    return 0;
}
```
**使用condition variable 搭配状态变量done和锁mutex 是必须的**  
但是上面情况并不是完美的比如下面的情况：  
>当主进程执行 thr_join() 的时候发现done==0,然后打算调用wait 去sleep的时候，主线程被中断，此后子线程执行。当子线程设置done=1和执行 sigin() 操作的时候发现没有在wait的线程，也就没有线程被唤醒了。当主线程再次执行的时候就会一直 wait and sleep forever

这就是race condition（竞态条件）

#### 场景二:**the producer/consumer or bounded-buffer problem** 
最简单的情况是
```c
//不足和问题：
//buffer 只保存一个int值
//buffer 写满的情况下还会继续写  buffer为空的时候还会继续读
int buffer;
int count = 0; // initially, empty
void put(int value) {
    assert(count == 0);
    count = 1;
    buffer = value;
}
int get() {
    assert(count == 1);
    count = 0;
    return buffer;
}

void *producer(void *arg) {
    int i;
    int loops = (int) arg;
    for (i = 0; i < loops; i++) {
        put(i);
    }
}
void *consumer(void *arg) {
    int i;
    while (1) {
        int tmp = get();
        printf("%d\n", tmp);
    }
}
```
上面的写法太简单且不实用，看一下面的写法

```c
//使用mutex lock 和 condition variable
int loops; // must initialize somewhere...
cond_t cond;
mutex_t mutex;
void *producer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        Pthread_mutex_lock(&mutex);              // p1
        if (count == 1)                          // p2
            Pthread_cond_wait(&cond, &mutex);   // p3
        put(i);                                 // p4
        Pthread_cond_signal(&cond);             // p5
        Pthread_mutex_unlock(&mutex);           // p6
    }
}

void *consumer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        Pthread_mutex_lock(&mutex);             // c1
        if (count == 0)                         // c2
            Pthread_cond_wait(&cond, &mutex);   // c3
        int tmp = get();                        // c4
        Pthread_cond_signal(&cond);             // c5
        Pthread_mutex_unlock(&mutex);           // c6
        printf("%d\n", tmp);
    }
}
```
上面这种写法在只有一个 producer 和 一个 consumer 的时候是可以的，但是如果有多个 consumer 的就已经不行了为什么这么说看下图 假设 Tc1 and Tc2  是consumer  Tp 是 producer

![1564114256399]({{ site.url }}/assets/img/1564114256399.png)



从上往下看执行顺序，1. Tc1拿到锁去读取buffer的时候发现buffer是空的，于是sleep，释放锁。2.这个时候Tp拿到这个锁， Tp获得锁开始写内容，Tp写完之后打算释放锁且唤醒等待的消费者Tc1（这个时候只有Tc1在等待），但是释放所得瞬间Tc2 拿到了锁, Tc2 执行拿走了Tc1等待的buffer 内容（这个时候buffer有内容，Tc2 不用wait），Tc1醒来的时候发现它的等的东西没了.

简单来说就是： producer 唤醒Tc1和  Tc1开始run的这中间，buffer的状态变了

怎么解决这个问题呢，就是Tc1唤醒的时候再检查一下count==1,具体就是**把if 变成while** 

```c
void *producer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        Pthread_mutex_lock(&mutex);              // p1
        while (count == 1)                          // p2
            Pthread_cond_wait(&cond, &mutex);   // p3
        put(i);                                 // p4
        Pthread_cond_signal(&cond);             // p5
        Pthread_mutex_unlock(&mutex);           // p6
    }
}

void *consumer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        Pthread_mutex_lock(&mutex);             // c1
        while (count == 0)                         // c2
            Pthread_cond_wait(&cond, &mutex);   // c3
        int tmp = get();                        // c4
        Pthread_cond_signal(&cond);             // c5
        Pthread_mutex_unlock(&mutex);           // c6
        printf("%d\n", tmp);
    }
}
```
解决了这个问题，其实还有另个一问题，看图  
![1564118292597]({{ site.url }}/assets/img/1564118292597.png)

这个问题，就是Tc1释放锁之后，Tc2获得了锁，然后两个消费者都sleep了（都执行到c3行代码），然后Tp获得锁往buffer写数据，写完之后唤醒Tc1，自己sleep, 释放锁。Tc1被唤醒获得然后拿走了buffer的数据，然后（c5）唤醒了Tc2, Tc2醒了之后发现没数据又sleep了，然后这三者都sleep了 

稍微想一下就知道问题在哪里，Tc1读完数据之后应该唤醒Tp让它继续写数据, 但是我们的condition variable 只有一个,所以Tc2被唤醒了。

所以我们再加一个conditon variable 来区分是唤醒producer  还是  consumer 

```c
cond_t empty, fill;
mutex_t mutex;
void *producer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        Pthread_mutex_lock(&mutex);
        while (count == 1)
            Pthread_cond_wait(&empty, &mutex);
        put(i);
        Pthread_cond_signal(&fill);
        Pthread_mutex_unlock(&mutex);
    }
}

void *consumer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        Pthread_mutex_lock(&mutex);
        while (count == 0)
            Pthread_cond_wait(&fill, &mutex);
        int tmp = get();
        Pthread_cond_signal(&empty);
        Pthread_mutex_unlock(&mutex);
        printf("%d\n", tmp);
    }
}
```
我们也得出一个原则，消费者consumer不能唤醒消费者，生产者producer也不能唤醒生产者  
以上是我们单元素buffer的producer&consumer问题，下面我们看看多元素buffer的情况  
正确的姿势:  
```c
int max;
int loops;
int *buffer;

int use_ptr = 0;
int fill_ptr = 0;
int num_full = 0;

pthread_cond_t empty = PTHREAD_COND_INITIALIZER;
pthread_cond_t fill = PTHREAD_COND_INITIALIZER;
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;

int consumers = 1;
int verbose = 1;


void do_fill(int value) {
    buffer[fill_ptr] = value;
    fill_ptr = (fill_ptr + 1) % max;
    num_full++;
}

int do_get() {
    int tmp = buffer[use_ptr];
    use_ptr = (use_ptr + 1) % max;
    num_full--;
    return tmp;
}

void *producer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        Mutex_lock(&m);            // p1
        while (num_full == max)     // p2
            Cond_wait(&empty, &m); // p3
        do_fill(i);                // p4
        Cond_signal(&fill);        // p5
        Mutex_unlock(&m);          // p6
    }

    // end case: put an end-of-production marker (-1)
    // into shared buffer, one per consumer
    for (i = 0; i < consumers; i++) {
        Mutex_lock(&m);
        while (num_full == max)
            Cond_wait(&empty, &m);
        do_fill(-1);
        Cond_signal(&fill);
        Mutex_unlock(&m);
    }

    return NULL;
}

void *consumer(void *arg) {
    int tmp = 0;
    // consumer: keep pulling data out of shared buffer
    // until you receive a -1 (end-of-production marker)
    while (tmp != -1) {
        Mutex_lock(&m);           // c1
        while (num_full == 0)      // c2
            Cond_wait(&fill, &m); // c3
        tmp = do_get();           // c4
        Cond_signal(&empty);      // c5
        Mutex_unlock(&m);         // c6
    }
    return NULL;
}
```







