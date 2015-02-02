---
layout: post
title: 使用C++11封装线程池ThreadPool
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
读本文之前，请务必阅读：
  [使用C++11的function/bind组件封装Thread以及回调函数的使用](http://www.cnblogs.com/inevermore/p/4038498.html)
  [Linux组件封装（五）一个生产者消费者问题示例](http://www.cnblogs.com/inevermore/p/4009719.html)
   
  线程池本质上是一个生产者消费者模型，所以请熟悉这篇文章：[Linux组件封装（五）一个生产者消费者问题示例](http://www.cnblogs.com/inevermore/p/4009719.html)。
  在ThreadPool中，物品为计算任务，消费者为pool内的线程，而生产者则是调用线程池的每个函数。
  搞清了这一点，我们很容易就需要得出，`ThreadPool需要一把互斥锁和两个同步变量，实现同步与互斥`。
  存储任务，当然需要一个任务队列。
  除此之外，我们还需要一系列的Thread，因为Thread无法复制，所以`我们使用unique_ptr作为一个中间层`。
  所以Thread的数据变量如下：
  

```Python
class ThreadPool : boost::noncopyable
{
public:
    typedef std::function<void ()> Task;

    ThreadPool(size_t queueSize, size_t threadsNum);
    ~ThreadPool();

    void start();
    void stop();

    void addTask(Task task); //C++11
    Task getTask();

    bool isStarted() const { return isStarted_; }

    void runInThread();

private:
    mutable MutexLock mutex_;
    Condition empty_;
    Condition full_;

    size_t queueSize_;
    std::queue<Task> queue_;

    const size_t threadsNum_;
    std::vector<std::unique_ptr<Thread> > threads_;
    bool isStarted_;
};
```
		

显然，我们使用了function，作为任务队列的任务元素。


 


构造函数的实现较简单，不过，之前务必注意元素的声明顺序与初始化列表的顺序相一致。




```C++
ThreadPool::ThreadPool(size_t queueSize, size_t threadsNum)
: empty_(mutex_),
  full_(mutex_),
  queueSize_(queueSize),
  threadsNum_(threadsNum),
  isStarted_(false)
{

}
```
		

添加和取走任务是生产者消费者模型最核心的部分，但是套路较为固定，如下：




```C++
void ThreadPool::addTask(Task task)
{
    MutexLockGuard lock(mutex_);
    while(queue_.size() >= queueSize_)
        empty_.wait();
    queue_.push(std::move(task));
    full_.notify();
}


ThreadPool::Task ThreadPool::getTask()
{
    MutexLockGuard lock(mutex_);
    while(queue_.empty())
        full_.wait();
    Task task = queue_.front();
    queue_.pop();
    empty_.notify();
    return task;
}
```
		

注意我们的addTask使用了C++11的move语义，在传入右值时，可以提高性能。


还有一些老生常谈的问题，例如：



  wait前加锁


  使用while循环判断wait条件（为什么？）



要想启动线程，需要给Thread提供一个回调函数，编写如下：




```C++
void ThreadPool::runInThread()
{
    while(1)
    {
        Task task(getTask());
        if(task)
            task();
    }
}
```
		

就是不停的取走任务，然后执行。


OK，有了线程的回调函数，那么我们可以编写start函数。




```C++
void ThreadPool::start()
{
    isStarted_ = true;
    //std::vector<std::unique<Thread> >
    for(size_t ix = 0; ix != threadsNum_; ++ix)
    {
        threads_.push_back(
            std::unique_ptr<Thread>(
                new Thread(
                    std::bind(&ThreadPool::runInThread, this))));
    }
    for(size_t ix = 0; ix != threadsNum_; ++ix)
    {
        threads_[ix]->start();
    }

}
```
		

这里较难理解的是线程的创建，Thread内存放的是std::unique_ptr<Thread>，而ptr的创建需要使用new动态创建Thread，Thread则需要在创建时，传入回调函数，我们采用bind适配runInThread的参数值。



###这里我们采用C++11的unique_ptr，成功实现vector无法存储Thread（为什么？）的问题


 


我们的第一个版本已经编写完毕了。


 



###添加stop功能


 


刚才的ThreadPool只能启动，无法stop，我们从几个方面着手，利用bool变量isStarted_，实现正确退出。


改动的有以下几点：


首先是Thread的回调函数不再是一个死循环，而是：




```C++
void ThreadPool::runInThread()
{
    while(isStarted_)
    {
        Task task(getTask());
        if(task)
            task();
    }
}
```
		



然后addTask和getTask，在while循环判断时，加入了bool变量：




```C++
void ThreadPool::addTask(Task task)
{
    MutexLockGuard lock(mutex_);
    while(queue_.size() >= queueSize_ && isStarted_)
        empty_.wait();

    if(!isStarted_)
        return;

    queue_.push(std::move(task));
    full_.notify();
}


ThreadPool::Task ThreadPool::getTask()
{
    MutexLockGuard lock(mutex_);
    while(queue_.empty() && isStarted_)
        full_.wait();

    if(!isStarted_) //线程池关闭
        return Task(); //空任务

    assert(!queue_.empty());
    Task task = queue_.front();
    queue_.pop();
    empty_.notify();
    return task;
}
```
		

这里注意，退出while循环后，需要再判断一次bool变量，因为未必是条件满足了，可能是线程池需要退出，调整了isStarted变量。


最后一个关键是我们的stop函数：


 




```C++
void ThreadPool::stop()
{
    if(isStarted_ == false)
        return;

    {
        MutexLockGuard lock(mutex_);
        isStarted_ = false;
        //清空任务
        while(!queue_.empty()) 
            queue_.pop();
    }
    full_.notifyAll(); //激活所有的线程
    empty_.notifyAll();
    
    for(size_t ix = 0; ix != threadsNum_; ++ix)
    {
        threads_[ix]->join();
    }
    threads_.clear();
}
```
		



这里有几个关键：


先将bool设置为false，然后调用notifyAll，`激活所有等待的线程`（为什么）。


 


最后我们`总结下ThreadPool关闭的流程`：



  1.isStarted设置为false




  2.加锁，清空队列




  3.发信号激活所有线程




  4.正在运行的Thread，执行到下一次循环，退出




  5.正在等待的线程被激活，然后while判断为false，执行到下一句，检查bool值，然后退出。




  6.主线程依次join每个线程。




  7.退出。



 


最后补充下析构函数的实现：




```C++
ThreadPool::~ThreadPool()
{
    if(isStarted_)
        stop();
}
```
		

 


完毕。

			