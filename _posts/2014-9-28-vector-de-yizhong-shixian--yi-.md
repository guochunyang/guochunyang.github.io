---
layout: post
title: Vector的一种实现（一）
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
 

注意几点：

分配内存不要使用new和delete，因为new的同时就把对象构造了，而我们需要的是原始内存。

所以应该使用标准库提供的allocator类来实现内存的控制。当然也可以重载operator new操作符，因为二者都是使用malloc作为底层实现，所以直接采用malloc也可以。

对象的复制必须使用系统提供的uninitialized_fill和uninitialized_copy，因为我们无法手工调用构造函数。

对于C++中的对象，除了POD之外，使用memcpy系列的函数是绝对错误的。

 

代码如下：



```Python
#ifndef VECTOR_H_
#define VECTOR_H_

#include <stddef.h>
#include <algorithm>
#include <memory>

template <typename T>
class Vector
{
public:
    typedef T *iterator;
    typedef const T *const_iterator;
    typedef size_t size_type;
    typedef T value_type;

    Vector() { create(); }
    explicit Vector(size_type n, const T &t = T())  { create(n, t); }
    Vector(const Vector &v) { create(v.begin(),  v.end()); }
    ~Vector() { uncreate(); }

    Vector &operator=(const Vector &other);
    T &operator[] (size_type i) { return data_[i]; }
    const T &operator[] (size_type i) const { return data_[i]; }

    void push_back(const T &t);

    size_type size() const { return avail_ - data_; }
    size_type capacity() const { return limit_ - data_; }

    iterator begin() { return data_; }
    const_iterator begin() const { return data_; }
    iterator end() { return avail_; }
    const_iterator end() const { return avail_; }

private:
    iterator data_; //首元素
    iterator avail_; //末尾元素的下一个位置
    iterator limit_; //内存的后面一个位置

    std::allocator<T> alloc_; //内存分配器

    void create();
    void create(size_type, const T &);
    void create(const_iterator, const_iterator);

    void uncreate();

    void grow();
    void uncheckedAppend(const T &);
};

template <typename T>
Vector<T> &Vector<T>::operator=(const Vector &rhs)
{
    if(this != &rhs)
    {
        uncreate(); //释放原来的内存
        create(rhs.begin(), rhs.end());
    }

    return *this;
}

template <typename T>
void Vector<T>::push_back(const T &t)
{
    if(avail_ == limit_)
    {
        grow();
    }
    uncheckedAppend(t);
}

template <typename T>
void Vector<T>::create()
{
    //分配空的数组
    data_ = avail_ = limit_ = 0;
}

template <typename T>
void Vector<T>::create(size_type n, const T &val)
{
    //分配原始内存
    data_ = alloc_.allocate(n);
    limit_ = avail_ = data_ + n;
    //向原始内存填充元素
    std::uninitialized_fill(data_, limit_, val);
}

template <typename T>
void Vector<T>::create(const_iterator i, const_iterator j)
{
    data_ = alloc_.allocate(j-i);
    limit_ = avail_ = std::uninitialized_copy(i, j, data_);
}

template <typename T>
void Vector<T>::uncreate()
{
    if(data_)
    {
        //逐个进行析构
        iterator it = avail_;
        while(it != data_)
        {
            alloc_.destroy(--it);
        }

        //真正的释放内存
        alloc_.deallocate(data_, limit_ - data_);
    }
    //重置指针
    data_ = limit_ = avail_ = 0;
}

template <typename T>
void Vector<T>::grow()
{
    //内存变为两倍
    size_type new_size = std::max(2 * (limit_ - data_), std::ptrdiff_t(1));
    //分配原始内存
    iterator new_data = alloc_.allocate(new_size);
    //复制元素
    iterator new_avail = std::uninitialized_copy(data_, avail_, new_data);

    uncreate(); //释放以前的内存，以及析构元素

    data_ = new_data;
    avail_ = new_avail;
    limit_ = data_ + new_size;
}

template <typename T>
void Vector<T>::uncheckedAppend(const T &val)
{
    alloc_.construct(avail_++, val);
}


#endif  /* VECTOR_H_ */
```
		
 

测试代码如下：



```C++
#include "Vector.hpp"
#include <iostream>
#include <string>
using namespace std;

int main(int argc, char const *argv[])
{
    Vector<string> vec(3, "hello");

    for(Vector<string>::const_iterator it = vec.begin();
        it != vec.end();
        ++it)
    {
        cout << *it << " ";
    }
    cout << endl;

    cout << "size = " << vec.size() << endl;
    cout << "capacity = " << vec.capacity() << endl;
    vec.push_back("foo");
    vec.push_back("bar");

    cout << "size = " << vec.size() << endl;
    cout << "capacity = " << vec.capacity() << endl;

    return 0;
}
```
		
			