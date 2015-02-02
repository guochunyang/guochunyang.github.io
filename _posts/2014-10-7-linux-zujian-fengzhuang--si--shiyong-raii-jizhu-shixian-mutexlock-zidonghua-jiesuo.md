---
layout: post
title: Linux组件封装（四）使用RAII技术实现MutexLock自动化解锁
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
我们不止一次写过这种代码：
  

```C++
{
    mutex_.lock();
    //XXX
    if(....)
        return;

    //XXX
    mutex_.unlock();
}
```
		



显然，这段代码中我们忘记了解锁。那么如何防止这种情况，我们采用和智能指针相同的策略，把加锁和解锁的过程封装在一个对象中。


实现`“对象生命期”等于“加锁周期”。`


代码如下：




```C++
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
```
		



这种把资源获取放在构造函数、资源释放放入析构函数中的做法，就是C++中的RAII技术，“`资源获取即初始化`”。它巧妙在C++中的栈对象是一定会析构的，所以资源一定会被释放。


这个类对于我们编写优雅的代码，好处是显而易见的，例如：




```C++
size_t Buffer::size() const
{
    mutex_.lock();
    int ret = queue_.size();
    mutex_.unlock();
    return queue_.size();
}
```
		
这段代码实在称不上美观，但是有了MutexLockGuard，我们可以写出：



```C++
size_t Buffer::size() const
{
    MutexLockGuard lock(mutex_);
    return queue_.size();
}
```
		

代码的美观性提高了许多。


当然，有一种使用方式是错误的，例如：




```C++
size_t Buffer::size() const
{
    MutexLockGuard(mutex_);
    return queue_.size();
}
```
		

这段代码的加锁周期仅限于那一行，为了防止错误使用，我们增加一个宏：




```Python
#define MutexLockGuard(m) "Error MutexLockGuard"
```
		
这样当错误使用的时候，会导致编译错误，使得我们早些发现问题。
			