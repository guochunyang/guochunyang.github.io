---
layout: post
title: 标准库Queue的实现
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
跟上篇实现stack的思路一致，我增加了一些成员函数模板，支持不同类型的Queue之间的复制和赋值。
  同时提供一个异常类。
  代码如下：
  

```Python
#ifndef QUEUE_HPP
#define QUEUE_HPP

#include "Exception.h"
#include <deque>

class EmptyQueueException : public Exception
{
public:
    EmptyQueueException() :Exception("read empty queue") { }
};

template <typename T, typename Container = std::deque<T> >
class Queue
{
public:
    typedef T value_type;
    typedef T& reference;
    typedef const T& const_reference;
    typedef Container container_type; //容器类型
    typedef EmptyQueueException exception_type; //异常类型
    typedef typename Container::size_type size_type;
    typedef Container &container_reference; //容器引用
    typedef const Container& const_container_reference;

    explicit Queue(const_container_reference cont = container_type()) :cont_(cont) { }

    template <typename T2, typename Container2>
    Queue<T, Container>(const Queue<T2, Container2> &other);

    template <typename T2, typename Container2>
    Queue<T, Container> &operator=(const Queue<T2, Container2> &other);

    void push(const value_type &val)
    {
        cont_.push_back(val);
    }

    void pop()
    {
        if(cont_.empty())
            throw exception_type();
        cont_.pop_front();
    }

    reference front()
    {
        if(cont_.empty())
            throw exception_type();
        return cont_.front();
    }
    const_reference front() const
    {
        if(cont_.empty())
            throw exception_type();
        return cont_.front();
    }
    reference back()
    {
        if(cont_.empty())
            throw exception_type();
        return cont_.back();
    }
    const_reference back() const
    {
        if(cont_.empty())
            throw exception_type();
        return cont_.back();
    }


    bool empty() const { return cont_.empty(); }
    size_type size() const { return cont_.size(); }

    //获取内部容器的引用
    const_container_reference get_container() const
    { return cont_; } 


    friend bool operator==(const Queue &a, const Queue &b)
    {
        return a.cont_ == b.cont_;
    }
    friend bool operator!=(const Queue &a, const Queue &b)
    {
        return a.cont_ != b.cont_;
    }
    friend bool operator<(const Queue &a, const Queue &b)
    {
        return a.cont_ < b.cont_;
    }
    friend bool operator>(const Queue &a, const Queue &b)
    {
        return a.cont_ > b.cont_;
    }
    friend bool operator<=(const Queue &a, const Queue &b)
    {
        return a.cont_ <= b.cont_;
    }
    friend bool operator>=(const Queue &a, const Queue &b)
    {
        return a.cont_ >= b.cont_;
    }


private:
    container_type cont_;
};


template <typename T, typename Container>
template <typename T2, typename Container2>
Queue<T, Container>::Queue(const Queue<T2, Container2> &other)
    :cont_(other.get_container().begin(), other.get_container().end())
{

}

template <typename T, typename Container>
template <typename T2, typename Container2>
Queue<T, Container> &Queue<T, Container>::operator=(const Queue<T2, Container2> &other)
{
    cont_.assign(other.get_container().begin(), other.get_container().end());
}


#endif //QUEUE_HPP
```
		

测试代码如下：




```C++
#include "Stack.hpp"
#include "Queue.hpp"
#include <iostream>
#include <string>
#include <vector>
#include <list>
#include <stdio.h>
using namespace std;

int main(int argc, char const *argv[])
{
   
    try
    {
        Queue<string, vector<string> > st;
        st.push("foo");
        st.push("bar");

        Queue<string, list<string> > st2(st);
        //st2 = st;

        while(!st2.empty())
        {
            cout << st2.front() << endl;
            st2.pop();
        }

        st2.pop(); //引发异常
    }
    catch (const Exception& ex)
    {
        fprintf(stderr, "reason: %s\n", ex.what());
        fprintf(stderr, "stack trace: %s\n", ex.stackTrace());
    }

    return 0;
}
```
		
			