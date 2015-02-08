---
layout: post
title: Linux组件封装（三）使用面向对象编程封装Thread
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
C++11提供了thread，但是过于复杂，我们还是倾向于在项目中编写自己的Thread。
  Posix Thread的使用这里不再赘述。
  重点是这个函数：
  

```C++
#include <pthread.h>
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                          void *(*start_routine) (void *), void *arg);
```
		

第三个参数是一个回调函数，该函数必须返回值为void*，而且只有一个参数，类型也是void*。


POSIX的thread默认是joinable，需要手工调用pthread_join函数来回收，`也可以调用pthread_detach将其变为detachable`，此时不需要手工回收线程。


下面介绍Thread的封装。


我们把Thread的声明先放在这里：




```Python
#ifndef THREAD_H_
#define THREAD_H_

#include "NonCopyable.h"
#include <pthread.h>

class Thread : NonCopyable
{
public:
    Thread();
    virtual ~Thread();

    void start();
    void join();

    virtual void run() = 0;

    pthread_t getThreadId() const
    { return threadId_; }

private:</pre>

  <pre>    //提供给pthread_create的第三个参数使用
    static void *runInThread(void *arg);
    
    pthread_t threadId_;
    //pid_t tid_; //进程标示
    bool isRunning_;
};

#endif //THREAD_H_
```
		

这里需要说明的是：



  首先，为了获得最干净的语义，`Thread应该是不可复制的`，所以需要继承NonCopyable。




  其次，为了调用pthread_create创建线程，我们往里面注册的不能是一个成员函数，因为成员函数含有一个隐式参数，导致函数的指针类型并不是void *(*start_routine) (void *)，所以我们`采用了static函数`。




  static函数无法访问某一对象的成员，所以我们在调用pthread_create时，`将this指针作为回调函数的参数`。



这里相关代码如下：




```C++
//static
void *Thread::runInThread(void *arg)
{
    Thread *pt = static_cast<Thread*>(arg);
    //pt->tid_ = syscall(SYS_gettid);
    pt->run();
    return NULL;
}
```
		
用户将自己的逻辑注册在run中就可以了。 


  这个Thread不提供detach函数，因为我们在析构函数中做了如下的处理，`如果Thread对象析构，线程还在运行，那么需要将Thread设置为detach状态`。





```C++
Thread::~Thread()
{
    if(isRunning_)
    {
        pthread_detach(threadId_);
    }
}
```
		

大部分逻辑都是固定的，用户只需要改变run里面的代码即可，于是我们`将run设置为纯虚函数，让用户继承Thread类`。



###所以析构函数为virtual


完整的CPP实现如下：




```C++
#include "Thread.h"
#include <assert.h>
#include <unistd.h>
#include "MutexLock.h" //TINY_CHECK

Thread::Thread()
    :threadId_(0),
     isRunning_(false)
{

}

Thread::~Thread()
{
    if(isRunning_)
    {
        TINY_CHECK(!pthread_detach(threadId_));
    }
}

//static
void *Thread::runInThread(void *arg)
{
    Thread *pt = static_cast<Thread*>(arg);
    //pt->tid_ = syscall(SYS_gettid);
    pt->run();
    return NULL;
}

void Thread::start()
{
    TINY_CHECK(!pthread_create(&threadId_, NULL, Thread::runInThread, this));
    isRunning_ = true;
}

void Thread::join()
{
    assert(isRunning_);
    TINY_CHECK(!pthread_join(threadId_, NULL));
    isRunning_ = false;
}
```
		

测试代码如下：采用继承的方式使用这个类。




```C++
#include "Thread.h"
#include <iostream>
#include <unistd.h>
using namespace std;

class MyThread : public Thread
{
public:
    void run()
    {
        cout << "foo" << endl;
    }
};

int main(int argc, char const *argv[])
{
    MyThread t;
    t.start();

    t.join();

    return 0;
}
```
		

NonCopyable类的定义如下：




```Python
#ifndef NONCOPYABLE_H
#define NONCOPYABLE_H


class NonCopyable //禁用值语义
{
public:
    NonCopyable() { }
    ~NonCopyable() { }
private:
    NonCopyable(const NonCopyable &);
    void operator=(const NonCopyable &);
};


#endif //NONCOPYABLE_H
```
		
			