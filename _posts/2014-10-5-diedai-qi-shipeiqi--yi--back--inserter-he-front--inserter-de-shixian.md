---
layout: post
title: 迭代器适配器（一）back_inserter和front_inserter的实现
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
本文讨论back_inserter和front_inserter的实现。

当我们调用copy函数的时候，要确保目标容器具有足够大的空间，例如：



```C++
//将other的所有元素拷贝到以coll.begin()为起始地址的位置
copy(other.begin(), other.end(), coll.begin());
```
		
如果之前没有为coll分配好内存，那么会引发越界错误。

如果我们无法提前预分配内存，那么怎么办？我们可以使用如下的代码：



```C++
//将other的所有元素拷贝到以coll.begin()为起始地址的位置
copy(other.begin(), other.end(), back_inserter(coll.begin()));
```
		
我们用了一个东西叫做back_inserter，它是一种插入迭代器（后面你会看到，它实际是个函数），那么插入迭代器是什么？

我们知道，迭代器用来实现容器操作的一种抽象，有了迭代器，那么我们遍历所有容器，采用的几乎都是同一种方式，换句话说，迭代器帮我们屏蔽了容器操作的细节。

对于元素的插入，不同的容器有不同的操作，例如push_back、insert等，插入迭代器就是帮我们屏蔽插入元素的细节，使得iter看起来总是指向一个&ldquo;可用的位置&rdquo;。

back_inserter的使用如下：



```C++
#include <iostream>
#include <string>
#include <vector>
#include <algorithm>
using namespace std;

template <typename T>
void printElems(const T &t, const string &s = "")
{
    cout << s << " ";
    for(typename T::const_iterator it = t.begin();
        it != t.end();
        ++it)
    {
        cout << *it << " ";
    }
    cout << endl;
}

int main(int argc, char const *argv[])
{
    vector<int> coll;
    back_insert_iterator<vector<int> > iter(coll);
    *iter = 1;
    iter++;
    *iter = 2;
    ++iter;
    *iter = 3;
    printElems(coll);


    back_inserter(coll) = 44;
    back_inserter(coll) = 55;

    printElems(coll);

    copy(coll.begin(), coll.end(), back_inserter(coll));
    printElems(coll);

    return 0;
}
```
		
可以看出，插入迭代器的使用很简易，而且不需要我们考虑内存的分配，因为迭代器内部帮我们处理了这些细节。正如前面所说，插入迭代器总是指向一块可用的位置，我们很快即将看到它的细节实现。

需要注意一下几点：


1.插入迭代器本质上是一种适配器，但是它看起来像一个迭代器，行为像一个迭代器，那么他就符合迭代器的定义。

2.插入迭代器的赋值，内部采用了插入元素的做法，可能调用容器的push_back push_front或者insert等。

3.插入迭代器的++操作，只是个幌子，但必不可少。以上面的copy为例，内部肯定调用了iter++，因为copy函数只是把它当做普通迭代器。

4.解引用操作同样也是幌子。


back_inserter和front_inserter实现代码如下



```Python
#ifndef ITERATOR_HPP
#define ITERATOR_HPP

template <typename Container>
class BackInsertIterator
{
public:
    typedef typename Container::value_type value_type;

    explicit BackInsertIterator(Container &cont) :cont_(cont) { }

    BackInsertIterator<Container> &operator=(const value_type &val)
    {
        cont_.insert(cont_.end(), val);
        return *this;
    }

    BackInsertIterator<Container> &operator*()
    {
        return *this;
    }

    BackInsertIterator<Container> &operator++()
    {
        return *this;
    }
    //iter++没有实质操作，所以也是返回引用
    BackInsertIterator<Container> &operator++(int)
    {
        return *this;
    }


private:
    Container &cont_;
};

template <typename Container>
BackInsertIterator<Container> backInserter(Container &c)
{
    return BackInsertIterator<Container>(c);
}

//FrontInsertIterator
template <typename Container>
class FrontInsertIterator
{
public:
    typedef typename Container::value_type value_type;

    explicit FrontInsertIterator(Container &cont) :cont_(cont) { }

    FrontInsertIterator<Container> &operator=(const value_type &val)
    {
        cont_.insert(cont_.begin(), val);
        return *this;
    }

    FrontInsertIterator<Container> &operator*()
    {
        return *this;
    }

    FrontInsertIterator<Container> &operator++()
    {
        return *this;
    }
    FrontInsertIterator<Container> &operator++(int)
    {
        return *this;
    }


private:
    Container &cont_;
};

template <typename Container>
FrontInsertIterator<Container> frontInserter(Container &c)
{
    return FrontInsertIterator<Container>(c);
}</pre>
<pre></pre>
<pre>#endif //ITERATOR_HPP
```
		
 

从上面的源码我们可以看到，二者插入均采用的insert操作。当然，调用push_back和push_front也是可以的。

			