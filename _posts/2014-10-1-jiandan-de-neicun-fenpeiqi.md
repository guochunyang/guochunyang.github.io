---
layout: post
title: 简单的内存分配器
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
采用自定义的operator运算符实现自己的内存分配策略，在某些时候可以提高程序的效率。
   
  C++中的new运算符，具体工作流程如下：
     1.调用operator new申请原始内存
    2.调用place new表达式，执行类的构造函数
    3.返回内存地址
   而delete操作符的工作是：
     1.调用对象的析构函数
    2.调用operator delete释放内存
   例如：
  

```C++
#include <iostream>
using namespace std;

class Test
{
public:
    Test() { cout << "Test" << endl; }
    ~Test() { cout << "~Test" << endl; }
};

int main(int argc, char const *argv[])
{

    //这里的pt指向的是原始内存
    Test *pt = static_cast<Test*>(operator new[] (5 * sizeof(Test)));
    
    for(int ix = 0; ix != 5; ++ix)
    {
        new (pt+ix)Test(); //调用定位new运算式 执行构造函数
    }

    for(int ix = 0; ix != 5; ++ix)
    {
        pt[ix].~Test(); //调用析构函数，但是并未释放内存
    }
    operator delete[] (pt); //释放内存

}
```
		



 


这里提供一个简单的内存分配器基类，凡是继承该类的class均具有自定义的operator new 和 operator delete


此示例来自《C++Primer》第四版


大概思想是用static变量维持一个链表，管理空闲的内存块。




```Python
#ifndef CACHED_OBJECT_HPP
#define CACHED_OBJECT_HPP

#include <memory>
#include <stdexcept>
#include <iostream> //debug

template <typename T>
class CachedObject
{
public:
    void *operator new(std::size_t);
    void operator delete(void *, std::size_t);
    virtual ~CachedObject() { }
protected:
    T *next_;
private:
    static void addToFreeList(T*);  //将内存块加入链表
    static std::allocator<T> alloc_;//内存分配器
    static T *freeStore_;           //空闲内存的链表
    static const std::size_t chunk_;//一次分配的块数
};

template <typename T> std::allocator<T> CachedObject<T>::alloc_;
template <typename T> T *CachedObject<T>::freeStore_ = NULL;
template <typename T> const std::size_t CachedObject<T>::chunk_ = 24;

template <typename T>
void *CachedObject<T>::operator new(std::size_t sz)
{
    if(sz != sizeof(T))
        throw std::runtime_error("CachedObject: wrong size object in operator new");
    
    std::cout << "operator new " << std::endl; //DEBUG

    //没有空闲内存
    if(freeStore_ == NULL)
    {
        T *array = alloc_.allocate(chunk_);
        for(std::size_t ix = 0; ix != chunk_; ++ix)
        {
            addToFreeList(&array[ix]);
        }
    }

    //取出一块内存，从链表取出第一个元素
    T *p = freeStore_;
    freeStore_ = freeStore_->CachedObject<T>::next_;
    return p;
}

template <typename T>
void CachedObject<T>::operator delete(void *p, std::size_t)
{
    std::cout << "operator delete " << std::endl; //DEBUG

    if(p != NULL)
        addToFreeList(static_cast<T*>(p));
}


template <typename T>
void CachedObject<T>::addToFreeList(T *p)
{
    //使用头插法
    p->CachedObject<T>::next_ = freeStore_;
    freeStore_ = p;
}

#endif /* CACHED_OBJECT_HPP */
```
		

每次执行new时，调用我们自定义的operator new去空闲链表中取出一块内存，如果链表为空，则执行真正的申请内存操作。


每次delete时，把内存归还给链表。


这样减少了每次new都去申请内存的开销。


测试代码如下：




```C++
#include "CachedObject.hpp"
#include <iostream>
using namespace std;

//使用继承的策略去使用这个内存分配器
class Test : public CachedObject<Test>
{

};

int main(int argc, char const *argv[])
{
    
    //调用自定义的new分配内存
    Test *pt = new Test;
    delete pt;

    //调用默认的new和delete
    pt = ::new Test;
    ::delete pt;

    //不会调用自定义的new和delete
    pt = new Test[10];
    delete[] pt; 

}
```
		
			