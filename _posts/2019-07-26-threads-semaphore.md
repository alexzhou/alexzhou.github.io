---
layout: post
title:  "Semaphores 信号量"
date: 2019-07-26 18:24:00
categories: OperateSystem
tags: [threads,concurrency,OSTEP笔记,Semaphores,信号量]
comments: true
---
Dijkstra 和他的同事研究完 lock 和 condition variable 发明了信号量 semaphore。 哪哪都能看到大佬的名字。  
信号量能以单个机制解决所有相关的多线程同步问题，可以把信号量当作locks 和  condition variable 使用。 

>信号量的定义:
>A semaphore is an object with an integer value that we can manipulate  
>with two routines; in the POSIX standard, these routines are sem wait()  
>and sem post()  

```c  
#include <semaphore.h
sem_t s;
sem_init(&s, 0, 1);
//第二个参数 0 表示只在单个进程的线程间共享信号量   1 多个进程中间的线程共享信号量
//第三个参数 1 信号量的int value 初始值
```

### semaphore信号量的用法  
用之前先确认三点：  
1. sem_wait() 如果信号量的值>0 先执行-1操作然后立刻return ; 如果信号量的值<=0 就阻塞调用它的线程
2. sem_post() 不像wait 操作那样需要特定的条件. 它直接把信号量的值+1, 唤醒正在等待的其中一个线程  
3. the value of the semaphore, when negative, is equal to the number of waiting threads  

```c
int sem_wait(sem_t *s) {
    //first: decrement the value of semaphore s by one
    //second: wait if value of semaphore s is negative
}
int sem_post(sem_t *s) {
    //increment the value of semaphore s by one 
    //if there are one or more threads waiting, wake one
}
```

#### 二进制信号量 Binary Semaphores (Locks) 
 
```c
sem_t m;
sem_init(&m, 0, X); // initialize to X; what should X be? 我们想要个锁 应该是 1
sem_wait(&m);
// critical section here
sem_post(&m);
```
#### Semaphores For Ordering
信号量在处理并发程序中的事件顺序也是很有用的 ， 以父线程等待子线程为例 ：  

```c
sem_t s;

void *child(void *arg) {
    printf("child\n");
    sem_post(&s); // signal here: child is done
    return NULL;
}
int main(int argc, char *argv[]) {
    //sem_init(&s, 0, X); // what should X be?  应该是0
    sem_init(&s, 0, 0)
    printf("parent: begin\n");
    pthread_t c;
    Pthread_create(&c, NULL, child, NULL);
    sem_wait(&s); // wait here for child  这行代吗和子线程谁先执行不影响  都是等待-1操作之后值变成非负数，
    printf("parent: end\n");
    return 0;
}
// A Parent Waiting For Its Chil
```

#### The Producer/Consumer (Bounded Buffer) Problem  

##### 1.只有一个buffer元素的情况

```c
int buffer[MAX];
int fill = 0;
int use = 0;
void put(int value) {
    buffer[fill] = value; // Line F1
    fill = (fill + 1) % MAX; // Line F2
}

int get() {
    int tmp = buffer[use]; // Line G1
    use = (use + 1) % MAX; // Line G2
    return tmp;
}

sem_t empty;
sem_t full;

void *producer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        sem_wait(&empty); // Line P1
        put(i); // Line P2
        sem_post(&full); // Line P3
    }
}

void *consumer(void *arg) {
    int i, tmp = 0;
    while (tmp != -1) {
        sem_wait(&full); // Line C1
        tmp = get(); // Line C2
        sem_post(&empty); // Line C3
        printf("%d\n", tmp);
    }
}

int main(int argc, char *argv[]) {
    // ...
    sem_init(&empty, 0, MAX); // MAX are empty
    sem_init(&full, 0, 0); // 0 are full
    // ...
}
```
在只有一个buffer元素情况下  多个consumer 和  producer 的情况也是可用的  
##### 2.多个buffer元素的情况  
多个元素的情况下 上面的代码 会出现race condition 的状况，导致会在一个producer覆盖另一个producer产生的数据  
改进的操作就是某个时段 只能有一个线程操作buffer (consumer 或者 producer) , 我们需要再加一个信号量当mutex lock（最前面说的binary semaphore）  
```c
void *producer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        sem_wait(&empty); // Line P1
        sem_wait(&mutex); // Line P1.5 (MUTEX   注意这个顺序  这个mutex要在empty 的后面，之前的大体逻辑不变 ，只是在读写的使用使用互斥锁
        put(i); // Line P2
        sem_post(&mutex); // Line P2.5 (AND HE
        sem_post(&full); // Line P3
    }
}

void *consumer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        sem_wait(&full); // Line C1
        sem_wait(&mutex); // Line C1.5 (MUTEX  注意这个顺序  这个mutex要在empty 的后面，之前的大体逻辑不变 ，只是在读写的使用使用互斥锁
        int tmp = get(); // Line C2
        sem_post(&mutex); // Line C2.5 (AND HE
        sem_post(&empty); // Line C3
        printf("%d\n", tmp);
    }
}
//如果调整两个信号量的顺序会怎样，死锁。 consumer 先进来获得锁， 但是要等待buffer有内容， 这个时候producer 需要获得锁些内容，但是consumer没释放锁  
```
#### Reader-Writer Locks
读写锁是一个经典的问题  
**写锁只允许一个writer获得，读锁允许多个reader 获得**  

```c

typedef struct _rwlock_t {
    sem_t lock; // binary semaphore (basic lock)
    sem_t writelock; // allow ONE writer/MANY readers
    int readers; // #readers in critical section
} rwlock_t; 

void rwlock_init(rwlock_t *rw) {
    rw->readers = 0;
    sem_init(&rw->lock, 0, 1);
    sem_init(&rw->writelock, 0, 1);
}

void rwlock_acquire_readlock(rwlock_t *rw) {
    sem_wait(&rw->lock);
    rw->readers++;
    if (rw->readers == 1) // first reader gets writelock
        sem_wait(&rw->writelock);
    sem_post(&rw->lock);
}

void rwlock_release_readlock(rwlock_t *rw) {
    sem_wait(&rw->lock);
    rw->readers--;
    if (rw->readers == 0) // last reader lets it go
        sem_post(&rw->writelock);
    sem_post(&rw->lock);
}

void rwlock_acquire_writelock(rwlock_t *rw) {
    sem_wait(&rw->writelock);
}

void rwlock_release_writelock(rwlock_t *rw) {
    sem_post(&rw->writelock);
}
```

#### The Dining Philosophers  
另一个经典问题：餐厅的哲学家  
假设有五个哲学家，五个叉子放在他们中间，他们有时候思考有时候吃饭，但是吃饭的时候要用左右两个叉子
```c
//假设p是线程的编号 0~4
while (1) {
    think();
    get_forks(p);
    eat();
    put_forks(p);
}

//左右两边叉子的编号  
int left(int p) { return p; }
int right(int p) { return (p + 1) % 5; }


//version 1

void get_forks(int p) {
    sem_wait(&forks[left(p)]);
    sem_wait(&forks[right(p)]);
}

void put_forks(int p) {
    sem_post(&forks[left(p)]);
    sem_post(&forks[right(p)]);
}
```   

哲学家想拿左边叉子的时候就是调用 left()，想拿右边的叉子就调用right() , version 1的代码的问题是有死锁：  
*每个人都拿起了左边的叉子 在等右边的叉子* 
所以我们需要有打破死锁的方法，打破死锁的方法有多种，我们看看version 2 Dijkstra大神的选择：  

```c
//让第五个哲学家p=4 不按照常理出牌 先拿右边的
void get_forks(int p) {
    if (p == 4) {
        sem_wait(&forks[right(p)]);
        sem_wait(&forks[left(p)]);
    } else {
        sem_wait(&forks[left(p)]);
        sem_wait(&forks[right(p)]);
    }
}
```

#### Thread Throttling  

线程节流的问题，CPU资源有限，太多线程会导致系统卡死，比如一开始要初始化一个最大可以使用的线程个数，避免耗尽资源

#### 信号量的实现  

创建一个我们自己的简单版本的信号量:  
我们只用一个lock 和一个 condition variable 外加一个状态变量来跟踪信号量的值  
```c
typedef struct __Zem_t {
    int value;
    pthread_cond_t cond;
    pthread_mutex_t lock;
} Zem_t;

// only one thread can call this
void Zem_init(Zem_t *s, int value) {
    s->value = value;
    Cond_init(&s->cond);
    Mutex_init(&s->lock);
}

void Zem_wait(Zem_t *s) {
    Mutex_lock(&s->lock);
    while (s->value <= 0)
        Cond_wait(&s->cond, &s->lock);
    s->value--;
    Mutex_unlock(&s->lock);
}

void Zem_post(Zem_t *s) {
    Mutex_lock(&s->lock);
    s->value++;
    Cond_signal(&s->cond);
    Mutex_unlock(&s->lock);
}
``` 
和Dijkstra 定义的信号量不同的是我们不维护信号量的值（Dijkstra定义的信号量的值的负数代表等待的线程数），我们定义的这个信号量的值实际用起来不会小于0  
PS：在信号量之外创建条件变量condition variable 是个棘手的问题， 很多有经验的程序员都栽了






