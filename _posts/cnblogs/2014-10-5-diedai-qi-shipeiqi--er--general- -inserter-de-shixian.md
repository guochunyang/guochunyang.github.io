---
layout: post
title: 迭代器适配器（二）general inserter的实现
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
上节我们实现了back_inserter和front_inserter，接下来是更为普通的插入迭代器，它允许用户指定插入位置。

实现代码如下：



```Python
#ifndef ITERATOR_HPP
#define ITERATOR_HPP

template <typename Container>
class InsertIterator
{
public:
    typedef typename Container::value_type value_type;
    typedef typename Container::iterator iterator;

    InsertIterator(Container &cont, iterator iter) :cont_(cont), iter_(iter) { }

    InsertIterator<Container> &operator=(const value_type &val)
    {
        cont_.insert(iter_, val);
        ++iter_;
        return *this;
    }

    InsertIterator<Container> &operator*()
    {
        return *this;
    }

    InsertIterator<Container> &operator++()
    {
        return *this;
    }
    InsertIterator<Container> &operator++(int)
    {
        return *this;
    }


private:
    Container &cont_;
    iterator iter_;
};

template <typename Container>
InsertIterator<Container> inserter(Container &c)
{
    return InsertIterator<Container>(c);
}


#endif //ITERATOR_HPP
```
		
可以看出，赋值操作使得内部存储的迭代器前移，而++操作同样什么都没有做。

或许我们想：可不可以在赋值操作符中不改变迭代器，而是到了++中改变？

答案是否定的。

设想，按照刚才设想的去实现，那么如果用户做了以下操作：



```C++
iter++;
iter++;
iter++;
*iter = 2;
```
		
那么我们没法保证最后一行时，iter指向的位置是可以插入元素的。

那么上面的写法为什么可行？原因是执行insert操作时，有个特殊位置为end()，它指向最后一个元素的下一个位置，也就是第一个非法的位置。这也是唯一一个合法的非法位置。

按照上面源码的实现，仅仅在赋值时迭代器前移，iter++无实质操作，而赋值时前移，iter实际指向了新的end()位置，就保证了无论用户怎么执行++操作，都丝毫不会影响iter的有效性。

测试代码如下：



```C++
#include "Iterator.hpp"
#include <iostream>
#include <string>
#include <vector>
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
    coll.push_back(12);
    coll.push_back(34);
    coll.push_back(32);
    printElems(coll);


    inserter(coll, coll.begin()) = 99; 
    inserter(coll, coll.begin()) = 88;

    printElems(coll);

    inserter(coll, coll.end()) = 34; 
    inserter(coll, coll.end()) = 21;

    printElems(coll);

    return 0;
}
```
		
			