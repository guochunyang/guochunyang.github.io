---
layout: post
title: 标准库Allocator(三)uninitialized_fill等函数的实现
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
前面我们使用了uninitialized_fill，来批量初始化某一段内存。
  下面提供三个函数的实现代码，这三个代码的共同点是：
     1.遇到错误，抛出异常
    2.出现异常时，把之前构造的对象全部销毁
   所以，这三个函数要么成功，要么无任何副作用。使用异常来通知使用者，所以在catch块中，处理完异常后要将异常再次向外抛出。
  

```Python
#ifndef MEMORY_HPP
#define MEMORY_HPP

#include <iterator>

template <typename ForwIter, typename T>
void uninitialized_fill(ForwIter beg, ForwIter end, const T & value)
{
    typedef typename std::iterator_traits<ForwIter>::value_type VT;
    ForwIter save(beg); //备份beg的初始值
    try
    {
        for(; beg != end ; ++beg)
        {
            new (static_cast<void *>(&*beg))VT(value);
        }
    }
    catch(...) //抛出异常
    {
        for(; save != beg; ++save)
        {
            save->~VT();    //逐个进行析构
        }
        throw; //将catch的异常再次抛出
    }
}


template <typename ForwIter, typename Size, typename T>
void uninitialized_fill_n(ForwIter beg, Size num, const T & value)
{
    typedef typename std::iterator_traits<ForwIter>::value_type VT;
    ForwIter save(beg);
    try
    {
        for(; num-- ; ++beg)
        {
            new (static_cast<void *>(&*beg))VT(value);
        }
    }
    catch(...)
    {
        for(; save != beg; ++save)
        {
            save->~VT();
        }
        throw;
    }
}

template <typename InputIter, typename ForwIter>
void uninitialized_copy(InputIter beg, InputIter end, ForwIter dest)
{
    typedef typename std::iterator_traits<ForwIter>::value_type VT;
    ForwIter save(dest);
    try
    {
        for(; beg != end ; ++beg, ++dest)
        {
            new (static_cast<void *>(&*dest))VT(*beg);
        }
    }
    catch(...)
    {
        for(; save != dest; ++save)
        {
            save->~VT();
        }
        throw;
    }
}

#endif /* MEMORY_HPP */
```
		

可以使用前面的代码自行测试。

			