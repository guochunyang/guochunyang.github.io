---
layout: post
title: 扩充STL-编写自己的迭代器
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
这里的迭代器能够与STL组件共同工作，是对STL的一种扩充。
   
  自定义迭代器必须提供iterator_traits的五种特性，分别是迭代器类型、元素类型、距离类型、指针类型与reference类型。
   
  这里我们继承标准库提供的iterator<>即可。
   
  代码如下： MyIterator.hpp 该迭代器针对于关联容器
  

```Python
#ifndef MYITERATOR_H_
#define MYITERATOR_H_

#include <iterator>

//必须提供五种类型，作为迭代器traits
template <typename Container>
class MyInsertIterator : public std::iterator<std::output_iterator_tag, 
                                            void, void, void, void>
{
    public:
        explicit MyInsertIterator(Container &c)
            :container_(c)
        {   }

        //将赋值转化为insert操作
        MyInsertIterator<Container> &operator=(const typename Container::value_type &value)
        {
            container_.insert(value); //针对的是关联容器 
            return *this;
        }

        //没有实际动作，起到掩饰的作用
        MyInsertIterator<Container> operator*()
        {
            return *this;
        }

        MyInsertIterator &operator++()
        {
            return *this;
        }

        MyInsertIterator &operator++(int)
        {
            return *this;
        }

    protected:
        Container &container_;
};


template <typename Container>
MyInsertIterator<Container> MyInsert(Container &c)
{
    return MyInsertIterator<Container>(c);
}


#endif /* MYITERATOR_H_ */
```
		

 


这里面重载的*和++没有实际操作，为的是起到掩饰的作用，从而仅仅返回自身引用。


 


注意=操作符执行了insert操作，所以当我们写下：


 




```C++
*iter = 3;
```
		



 


时，自动将3插入至容器中。


 


我们还提供了一个MyInsert函数，用于快速生成迭代器对象，于是我们可以这样使用：




```C++
MyInsert(coll) = 55;
```
		



 


我们只需改动代码中的
  

```C++
container_.insert(value); //针对的是关联容器
```
		



如果改为push_back就变成了back_inserter，如果调用push_front则成为front_inserter


 


测试代码如下：




```C++
#include "MyIterator.hpp"
#include <set>
#include <string>
#include <iostream>
using namespace std;

template <typename CONT>
void print(const CONT &s)
{
    for(typename CONT::const_iterator it = s.begin();
        it != s.end();
        ++it)
    {
        cout << *it << " ";
    }
    cout << endl;
}

int main(int argc, char const *argv[])
{
    set<int> coll;

    MyInsertIterator<set<int> > iter(coll);

    *iter = 1;
    iter++;
    *iter = 2;
    iter++;
    *iter = 3;

    print(coll);

    MyInsert(coll) = 44;
    MyInsert(coll) = 55;


    print(coll);

    int vals[] = {33, 67, -4, 13, 5, 2};
    int size = sizeof(vals) / sizeof(vals[0]);
    copy(vals, vals + size, MyInsert(coll));

    print(coll);

    return 0;
}
```
		

 


结果为：




```C++
1 2 3 
1 2 3 44 55 
-4 1 2 3 5 13 33 44 55 67
```
		
			