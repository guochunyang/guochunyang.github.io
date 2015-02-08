---
layout: post
title: Linux组件封装（二）中条件变量Condition的封装
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
条件变量主要用于实现线程之间的协作关系。
  pthread_cond_t常用的操作有：
  

```C++
int pthread_cond_init(pthread_cond_t *cond, pthread_condattr_t *cond_attr);
int pthread_cond_signal(pthread_cond_t *cond);
int pthread_cond_broadcast(pthread_cond_t *cond);
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
int pthread_cond_destroy(pthread_cond_t *cond);
```
		

我先将Condition类的声明放在这里：




```Python
#ifndef CONDITION_H_
#define CONDITION_H_

#include "NonCopyable.h"
#include <pthread.h>

class MutexLock;

class Condition : NonCopyable
{
public:
    Condition(MutexLock &mutex);
    ~Condition();

    void wait();
    void notify();
    void notifyAll();

private:
    pthread_cond_t cond_;
    MutexLock &mutex_;
};

#endif //CONDITION_H_
```
		

这里注意几点：



  1.wait必须在加锁的条件下调用。


  2.notify一次唤醒一个线程，`通常用来通知资源可用`，


  3.notifyAll一次通知多个线程，`通常用来通知状态的改变`。滥用broadcast会导致“惊群”问题。




###使用wait必须采用while判断



  a) 如果采用if，最多判断一次。




  b) 线程A等待数据，阻塞在full上，那么当另一个线程放入产品时，通知A去拿数据，此时另一个线程B抢到锁，直接进入临界区，取走资源。A重新抢到锁，（因为采用的是if，所以不会判断第二次）进去临界区时，已经没有资源。




  c)防止broadcast的干扰，`如果获得一个资源，使用broadcast会唤醒所有等待的线程，那么多个线程被唤醒，但最终只有一个能拿到资源，这就是所谓的“惊群效应”。`



cpp代码实现如下：




```C++
#include "Condition.h"
#include "MutexLock.h"
#include <assert.h>

Condition::Condition(MutexLock &mutex)
    :mutex_(mutex)
{
    TINY_CHECK(!pthread_cond_init(&cond_, NULL));
}

Condition::~Condition()
{
    TINY_CHECK(!pthread_cond_destroy(&cond_));
}


void Condition::wait()
{
    assert(mutex_.isLocking()); //wait前必须上锁
    TINY_CHECK(!pthread_cond_wait(&cond_, mutex_.getMutexPtr()));
    mutex_.restoreMutexStatus(); //还原状态
}

void Condition::notify()
{
    TINY_CHECK(!pthread_cond_signal(&cond_));
}

void Condition::notifyAll()
{
    TINY_CHECK(!pthread_cond_broadcast(&cond_));
}
```
		



注意wait后面调用了restoreMutexStatus将mutex的状态置为true。

			