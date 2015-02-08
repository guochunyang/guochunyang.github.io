---
layout: post
title: 使用C++11的function/bind组件封装Thread以及回调函数的使用
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
之前在[http://www.cnblogs.com/inevermore/p/4008572.html](http://www.cnblogs.com/inevermore/p/4008572.html)中采用面向对象的方式，封装了Posix的线程，那里采用的是虚函数+继承的方式，用户通过重写Thread基类的run方法，传入自己的用户逻辑。
   
  现在我们采用C++11的function，将函数作为Thread类的成员，用户只需要将function对象传入线程即可，所以Thread的声明中，应该含有一个function成员变量。
  类的声明如下：
  

```Python
#ifndef THREAD_H_
#define THREAD_H_

#include <boost/noncopyable.hpp>
#include <functional>
#include <pthread.h>

class Thread : boost::noncopyable
{
public:
    typedef std::function<void ()> ThreadCallback;

    Thread(ThreadCallback callback);
    ~Thread();

    void start();
    void join();

    static void *runInThread(void *);

private:
    pthread_t threadId_;
    bool isRunning_;
    ThreadCallback callback_; //回调函数
};



#endif //THREAD_H_
```
		

那么如何开启线程？思路与之前一致，写一个static函数，用户pthread_create的第三个参数，this作为最后一个参数即可。




```C++
void Thread::start()
{
    pthread_create(&threadId_, NULL, runInThread, this);
    isRunning_ = true;
}
```
		

 



###回调函数


 


注意在这种封装方式中，我们采用了回调函数。回调函数与普通函数的区别就是，普通函数写完由我们自己直接调用，函数调用是一种不断往上堆积的方式，而回调函数通常是我们把某一个函数传入一个“盒子”，由该盒子内的机制来调用它。


在这个例子里面，我们将function传入Thread，当Thread启动的时候，由Thread去执行function对象。


`在win32编程中大量用到这种机制`，我们为鼠标单击、双击等事件编写相应的函数，然后将其注册给windows系统，然后系统在我们触发各种事件的时候，根据事件的类型，调用相应的构造函数。


关于回调函数，可以参考：<a title="http://www.zhihu.com/question/19801131" href="http://www.zhihu.com/question/19801131">http://www.zhihu.com/question/19801131</a>


以后有时间，再专门总结下回调函数。


 


完整的cpp如下：




```C++
#include "Thread.h"

Thread::Thread(ThreadCallback callback)
: threadId_(0),
  isRunning_(false),
  callback_(std::move(callback))
{

}
    
Thread::~Thread()
{
    if(isRunning_)
    {
        //detach
        pthread_detach(threadId_);
    }
}

void Thread::start()
{
    pthread_create(&threadId_, NULL, runInThread, this);
    isRunning_ = true;
}
void Thread::join()
{
    pthread_join(threadId_, NULL);
    isRunning_ = false;
}

void *Thread::runInThread(void *arg)
{
    Thread *pt = static_cast<Thread*>(arg);
    pt->callback_(); //调用回调函数

    return NULL;
}
```
		

 


这个线程的使用方式有三种：


一是将普通函数作为回调函数




```C++
void foo()
{
    while(1)
    {
        printf("foo\n");
        sleep(1);
    }
}

int main(int argc, char const *argv[])
{
    Thread t(&foo);

    t.start();
    t.join();

    return 0;
}
```
		

 


二是采用类的成员函数作为回调函数：




```C++
class Foo
{
public:
    void foo(int i)
    {
        while(1)
        {
            printf("foo %d\n", i++);
            sleep(1);
        }
    }
};


int main(int argc, char const *argv[])
{
    Foo f;
    int i = 34;
    Thread t(bind(&Foo::foo, &f, i));

    t.start();
    t.join();

    return 0;
}
```
		

最后一种是组合一个新的线程类，注意这里采用的是`类的组合`：




```C++
class Foo
{
public:

    Foo()
    : thread_(bind(&Foo::foo, this))
    {
    }

    void start()
    {
        thread_.start();
        thread_.join();
    }

    void foo()
    {
        while(1)
        {
            printf("foo\n");
            sleep(1);
        }
    }
private:
    Thread thread_;
};

int main(int argc, char const *argv[])
{
    Foo f;
    f.start();

    return 0;
}
```
		

有些复杂的类，还需要将三种方式加以整合，例如后面要谈到的TimerThread，里面含有一个Thread和Timer，用户将逻辑注册给Timer，然后Timer的start函数注册给Thread。


这种方式的Thread，使用灵活性相对于面向对象的风格，提高了很多。


 





###基于对象和面向对象


 


这里总结几点：



  面向对象依靠的是`虚函数+继承`，用户通过重写基类的虚函数，实现自己的逻辑。




  基于对象，`依赖类的组合，使用function和bind实现委托机制，更加依赖于回调函数`传入逻辑。

			