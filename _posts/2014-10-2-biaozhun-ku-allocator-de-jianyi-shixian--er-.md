---
layout: post
title: 标准库Allocator的简易实现（二）
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
自己实现Allocator并不难，其实只需要改变allocate和deallocate，来实现自己的内存分配策略。
   
  下面是一个std::allocator的模拟实现
  

```Python
#ifndef ALLOCATOR_HPP
#define ALLOCATOR_HPP

#include <stddef.h>
#include <limits>

template <typename T>
class Allocator
{
public:
    typedef size_t size_type;
    typedef ptrdiff_t difference_type;
    typedef T*  pointer;
    typedef const T* const_pointer;
    typedef T& reference;
    typedef const T& const_reference;
    typedef T value_type;

    //Allocator::rebind<T2>::other
    template <typename V>
    struct rebind
    {
        typedef Allocator<V> other;
    };

    pointer address(reference value) const { return &value; }
    const_pointer address(const_reference value) const { return &value; }

    Allocator() throw() {   }
    Allocator(const Allocator &) throw() {  }
    //不同类型的allcator可以相互复制
    template <typename V> Allocator(const Allocator<V> &other) { } 
    ~Allocator() throw() {  }

    //最多可以分配的数目
    size_type max_size() const throw()
    { return std::numeric_limits<size_type>::max() / sizeof(T); }

    //分配内存，返回该类型的指针
    pointer allocate(size_type num)
    { return (pointer)(::operator new(num * sizeof(T))); }

    //执行构造函数，构建一个对象
    void construct(pointer p, const T &value)
    { new ((void*)p) T(value); }

    //销毁对象
    void destroy(pointer p)
    { p->~T(); }

    //释放内存
    void deallocate(pointer p, size_type num)
    { ::operator delete((void *)p); }
};

//这两个运算符不需要friend声明
template <typename T, typename V>
bool operator==(const Allocator<T> &, const Allocator<V> &) throw()
{ return true; }

template <typename T, typename V>
bool operator!=(const Allocator<T> &, const Allocator<V> &) throw()
{ return false; }


#endif
```
		



这里注意rebind的实现，如果需要使用Test的分配器分配其他类型，就可以这样：




```C++
Allocator<Test>::rebind<Test2>::other alloc;
```
		

测试代码如下：




```C++
#include "Allocator.hpp"
#include <string>
#include <vector>
using namespace std;


int main(int argc, char const *argv[])
{
    vector<string, Allocator<string> > vec(10, "haha");

    vec.push_back("foo");
    vec.push_back("bar");

    //Allocator<Test>::rebind<Test2>::other alloc;

    return 0;
}
```
		
			