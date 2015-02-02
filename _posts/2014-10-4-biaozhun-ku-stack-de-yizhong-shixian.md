---
layout: post
title: 标准库Stack的一种实现
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
本文实现了STL中stack的大部分功能，同时添加了一些功能。
  注意以下几点：
     1.Stack是一种适配器，底层以vector、list、deque等实现
    2.Stack不含有迭代器
   在本例中，我添加了几项功能，包括不同类型stack之间的复制和赋值功能，可以实现诸如Stack<int, vector<int> >和Stack<double, list<double> >之间的复制和赋值，这主要依靠成员函数模板来实现。
  为了更方便的实现以上功能，我添加了一个函数：
  

```C++
const_container_reference get_container() const
```
		



来获取内部容器的引用。


此外，标准库的stack不检查越界行为，我为stack添加了异常处理，当栈空时，执行pop或者top会抛出异常。这个异常类继承自Exception（见上篇文章），用来标示栈空。


详细代码如下：Exception的实现见：[借助backtrace和demangle实现异常类Exception](http://www.cnblogs.com/inevermore/p/4005489.html)




```Python
#ifndef STACK_HPP_
#define STACK_HPP_

#include "Exception.h"
#include <deque>

//栈空引发的异常
class EmptyStackException : public Exception
{
public:
    EmptyStackException() :Exception("read empty stack") { }
};


template <typename T, typename Container = std::deque<T> >
class Stack
{
public:
    typedef T value_type;
    typedef T& reference;
    typedef const T& const_reference;
    typedef Container container_type; //容器类型
    typedef EmptyStackException exception_type; //异常类型
    typedef typename Container::size_type size_type;
    typedef Container &container_reference; //容器引用
    typedef const Container& const_container_reference;

    explicit Stack(const container_type &cont = container_type()) :cont_(cont) { }</pre>

  <pre>
    //不同类型间实现复制
    template <typename T2, typename Container2>
    Stack<T, Container>(const Stack<T2, Container2> &s); 

    //不同类型间进行赋值
    template <typename T2, typename Container2>
    Stack<T, Container> &operator=(const Stack<T2, Container2> &s);

    void push(const value_type &val) { cont_.push_back(val); }
    void pop() 
    { 
        if(cont_.empty())
            throw exception_type();
        cont_.pop_back(); 
    }

    reference top()
    {
        if(cont_.empty())
            throw exception_type();
        return cont_.back();
    } 
    const_reference top() const
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


    friend bool operator==(const Stack &a, const Stack &b)
    {
        return a.cont_ == b.cont_;
    }
    friend bool operator!=(const Stack &a, const Stack &b)
    {
        return a.cont_ != b.cont_;
    }
    friend bool operator<(const Stack &a, const Stack &b)
    {
        return a.cont_ < b.cont_;
    }
    friend bool operator>(const Stack &a, const Stack &b)
    {
        return a.cont_ > b.cont_;
    }
    friend bool operator<=(const Stack &a, const Stack &b)
    {
        return a.cont_ <= b.cont_;
    }
    friend bool operator>=(const Stack &a, const Stack &b)
    {
        return a.cont_ >= b.cont_;
    }

private:
    Container cont_;
};

template <typename T, typename Container>
template <typename T2, typename Container2>
Stack<T, Container>::Stack(const Stack<T2, Container2> &s)
    :cont_(s.get_container().begin(), s.get_container().end())
{

}

template <typename T, typename Container>
template <typename T2, typename Container2>
Stack<T, Container> &Stack<T, Container>::operator=(const Stack<T2, Container2> &s)
{
    if((void*)this != (void*)&s) 
    {
        cont_.assign(s.get_container().begin(), s.get_container().end());
    }
    
    return *this;
}


#endif /* STACK_HPP_ */
```
		

测试代码如下：




```C++
#include "Stack.hpp"
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
        Stack<string, vector<string> > st;
        st.push("foo");
        st.push("bar");

        Stack<string, list<string> > st2(st);
        //st2 = st;

        while(!st2.empty())
        {
            cout << st2.top() << endl;
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
		
			