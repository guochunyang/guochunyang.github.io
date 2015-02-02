---
layout: post
title: Linux组件封装（一）中互斥锁MutexLock的封装
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
本文对Linux中的pthread_mutex_t做一个简易的封装。
  互斥锁主要用于互斥，互斥是一种竞争关系，主要是某一个系统资源或一段代码，一次做多被一个线程访问。
  条件变量主要用于同步，用于协调线程之间的关系，是一种合作关系。
  Linux中互斥锁的用法很简单，最常用的是以下的几个函数：
  

```C++
int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *mutexattr);
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
int pthread_mutex_destroy(pthread_mutex_t *mutex);
```
		

我们利用RAII技术，将mutex的初始化和销毁放在构造函数和析构函数中。


这里注意，pthread系列的函数都是成功时返回0，我采用一个轻量级的检查手段，来判断处理错误。




```Python
//用于pthread系列函数的返回值检查
#define TINY_CHECK(exp) \
    if(!(exp)) \
    {   \
        fprintf(stderr, "File:%s, Line:%d Exp:[" #exp "] is true, abort.\n", __FILE__, __LINE__); abort();\
    }
```
		

代码如下：




```Python
#ifndef MUTEXLOCK_H
#define MUTEXLOCK_H

#include "NonCopyable.h"
//#include <boost/noncopyable.hpp>
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

//用于pthread系列函数的返回值检查
#define TINY_CHECK(exp) \
    if(!(exp)) \
    {   \
        fprintf(stderr, "File:%s, Line:%d Exp:[" #exp "] is true, abort.\n", __FILE__, __LINE__); abort();\
    }


class MutexLock : NonCopyable
{
    friend class Condition;
public:
    MutexLock();
    ~MutexLock();
    void lock();
    void unlock();

    bool isLocking() const { return isLocking_; }
    pthread_mutex_t *getMutexPtr() //思考为什么不是const
    { return &mutex_; } 

private:
    void restoreMutexStatus() //提供给Condition的wait使用
    { isLocking_ = true; }


    pthread_mutex_t mutex_;
    bool isLocking_; //是否上锁
};


class MutexLockGuard : NonCopyable
{
public:
    MutexLockGuard(MutexLock &mutex) :mutex_(mutex)
    { mutex_.lock(); }
    ~MutexLockGuard()
    { mutex_.unlock(); }
private:
    MutexLock &mutex_;
};


#endif //MUTEXLOCK_H
```
		

cpp文件：




```C++
#include "MutexLock.h"
#include <assert.h>

MutexLock::MutexLock()
    :isLocking_(false)
{
    TINY_CHECK(!pthread_mutex_init(&mutex_, NULL));
}

MutexLock::~MutexLock()
{
    assert(!isLocking());//确保解锁
    TINY_CHECK(!pthread_mutex_destroy(&mutex_));
}


void MutexLock::lock()
{
    
    TINY_CHECK(!pthread_mutex_lock(&mutex_));
    isLocking_ = true;
}

void MutexLock::unlock()
{
    isLocking_ = false;
    TINY_CHECK(!pthread_mutex_unlock(&mutex_));
}
```
		

后面我们可以继续使用RAII，将lock和unlock封装在同一个类中。

			