---
layout: post
title: Linux组件封装（五）一个生产者消费者问题示例
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
生产者消费者问题是计算机中一类重要的模型，主要描述的是：生产者往缓冲区中放入产品、消费者取走产品。生产者和消费者指的可以是线程也可以是进程。
  生产者消费者问题的难点在于：
     为了缓冲区数据的安全性，一次只允许一个线程进入缓冲区，它就是所谓的临界资源。
      生产者往缓冲区放物品时，如果缓冲区已满，那么需要等待，一直到消费者取走产品为止。
      消费者取走产品时，如果没有物品，需要等待，一直到有生产者放入为止。
   第一个问题属于互斥问题，我们需要使用一把`互斥锁`，来实现对缓冲区的安全访问。
  后两个属于`同步问题，两类线程相互协作`，需要两个`条件变量`，一个用于通知生产者放入产品，一个用来通知消费者取走物品。
  生产者线程的大概流程是：
     1.加锁
    2.如果缓冲区已满，则等待。
    3.放入物品
    4.解锁
    5.通知消费者，可以取走产品
   消费者的逻辑恰好相反：
     1.加锁
    2.缓冲区为空，等待
    3.取走物品
    4.解锁
    5.通知生产者，可以放入物品
   我们设计出一个缓冲区：
  

```Python
#ifndef BUFFER_H_
#define BUFFER_H_

#include "NonCopyable.h"
#include "MutexLock.h"
#include "Condition.h"
#include <queue>

class Buffer : NonCopyable
{
public:
    Buffer(size_t size);
    void push(int val);
    int pop();

    bool empty() const;
    size_t size() const;

private:
    mutable MutexLock mutex_;
    Condition full_;
    Condition empty_;

    size_t size_; //缓冲区的大小
    std::queue<int> queue_;
};

#endif //BUFFER_H_
```
		



这里注意，我们把同步与互斥的操作都放入Buffer中，使得生产者和消费者线程不必考虑其中细节，这符号软件设计的`“高内聚、低耦合”`原则。


还有一点，mutex被声明为mutable类型，意味着mutex在const函数中仍然可以被改变，这是符合程序的逻辑的。`把mutex声明为mutable，是一种标准实践。`


重点是push和pop的实现：




```C++
void Buffer::push(int val)
{
    //lock
    //wait
    //push
    //notify
    //lock
    {
        MutexLockGuard lock(mutex_);
        while(queue_.size() >= size_)
            empty_.wait();
        queue_.push(val);
    }
    full_.notify();
}

int Buffer::pop()
{
    int temp = 0;
    {
        MutexLockGuard lock(mutex_);
        while(queue_.empty())
            full_.wait();
        temp = queue_.front();
        queue_.pop();
    }
    empty_.notify();

    return temp;
}
```
		

这里注意：



  1.`条件变量的等待必须使用while`，这是一种最佳实践，原因可见Condition的封装[Linux组件封装（二）中条件变量Condition的封装](http://www.cnblogs.com/inevermore/p/4008397.html)。


  2.可以先notify再解锁，也可以先解锁。`不过推荐先解锁`，原因是如果先notify，唤醒一个线程B，但是还未解锁，此时如果线程切换至刚唤醒的线程B，B马上尝试lock，但是肯定失败，然后阻塞，`这增加了一次线程切换的开销`。



我们还可以继续封装，将缓冲区与多个生产者、消费者封装成一个车间类，如下：




```Python
#ifndef WORKSHOP_H_
#define WORKSHOP_H_

#include "NonCopyable.h"
#include "Buffer.h"
#include <vector>

class ProducerThread;
class ConsumerThread;

class WorkShop : NonCopyable
{
public:
    WorkShop(size_t bufferSize, 
             size_t producerSize, 
             size_t consumerSize);
    ~WorkShop();

    void startWorking();
private:
    size_t bufferSize_;
    Buffer buffer_;

    size_t producerSize_;
    size_t consumerSize_;
    std::vector<ProducerThread*> producers_;
    std::vector<ConsumerThread*> consumers_;
};


#endif //WORKSHOP_H_
```
		

这样就可以方便的指定线程的数目。


 


完整的项目代码请参见这里：[生产者消费者完整代码](https://bitbucket.org/gchunyang/producerandconsumer/src)``

			