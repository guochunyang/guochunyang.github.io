---
layout: post
title: 模板：什么是Traits
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
Traits不是一种语法特性，而是一种模板编程技巧。Traits在C++标准库，尤其是STL中，有着不可替代的作用。
   
  
###如何在编译期间区分类型
   
  下面我们看一个实例，有四个类，Farm、Worker、Teacher和Doctor，我们需要区分他们是脑力劳动者还是体力劳动者。以便于做出不同的行动。
  这里的问题在于，我们需要`为两种类型提供一个统一的接口，但是对于不同的类型，必须做出不同的实现`。
  我们不希望写两个函数，然后让用户去区分。
  于是我们借助了函数重载，在每个类的内部内置一个work_type，然后根据每个类的word_type，借助强大的函数重载机制，实现了编译期的类型区分，也就是编译期多态。
  代码如下：
  

```Python
#include <iostream>
using namespace std;

//两个标签类
struct brain_worker {}; //脑力劳动
struct physical_worker {}; //体力劳动

class Worker
{
public:
    typedef physical_worker worker_type;
};

class Farmer
{
public:
    typedef physical_worker worker_type;
};

class Teacher
{
public:
    typedef brain_worker worker_type;
};

class Doctor
{
public:
    typedef brain_worker worker_type;
};

template <typename T>
void __distinction(const T &t, brain_worker)
{
    cout << "脑力劳动者" << endl;
}

template <typename T>
void __distinction(const T &t, physical_worker)
{
    cout << "体力劳动者" << endl;
}

template <typename T>
void distinction(const T &t)
{
    typename T::worker_type _type; //为了实现重载
    __distinction(t, _type);
}

int main(int argc, char const *argv[])
{
    Worker w;
    distinction(w);
    Farmer f;
    distinction(f);
    Teacher t;
    distinction(t);
    Doctor d;
    distinction(d);
    return 0;
}
```
		

在distinction函数中，我们先从类型中提取出worker_type，然后根据它的类型，选取不同的实现。


 


问题来了，如果不在类中内置worker_type，或者有的类已经写好了，无法更改了，那么怎么办？


 



###使用Traits


 


我们的解决方案是，借助一种叫做traits的技巧。


我们写一个模板类，但是不提供任何实现：




```C++
//类型traits 
template <typename T>
class TypeTraits;
```
		

然后我们`为每个类型提供一个模板特化`：




```Python
//为每个类型提供一个特化版本
template <>
class TypeTraits<Worker>
{
public:
    typedef physical_worker worker_type;
};

template <>
class TypeTraits<Farmer>
{
public:
    typedef physical_worker worker_type;
};

template <>
class TypeTraits<Teacher>
{
public:
    typedef brain_worker worker_type;
};

template <>
class TypeTraits<Doctor>
{
public:
    typedef brain_worker worker_type;
};
```
		



然后在distinction函数中，`不再是直接寻找内置类型，而是通过traits抽取出来`。




```C++
template <typename T>
void distinction(const T &t)
{
    //typename T::worker_type _type;
    typename TypeTraits<T>::worker_type _type;
    __distinction(t, _type);
}
```
		

 


上面两种方式的`本质区别在于，第一种是在class的内部内置type，第二种则是在类的外部，使用模板特化，class本身对于type并不知情。`


 



###两种方式结合


 


上面我们实现了目的，类中没有work_type时，也可以正常运行，但是模板特化相对于内置类型，还是麻烦了一些。


于是，我们仍然使用内置类型，也仍然使用traits抽取work_type，方法就是`为TypeTraits提供一个默认实现，默认去使用内置类型，``把二者结合起来。`


这样我们去使用TypeTraits<T>::worker_type时，`有内置类型的就使用默认实现，无内置类型的就需要提供特化版本`。




```Python
class Worker
{
public:
    typedef physical_worker worker_type;
};

class Farmer
{
public:
    typedef physical_worker worker_type;
};

class Teacher
{
public:
    typedef brain_worker worker_type;
};

class Doctor
{
public:
    typedef brain_worker worker_type;
};


//类型traits 
template <typename T>
class TypeTraits
{
public:
    typedef typename T::worker_type worker_type;
};
```
		

OK，我们现在想添加一个新的class，于是我们有两种选择，



  一是在class的内部内置work_type，通过traits的默认实现去抽取type。




  一种是不内置work_type，而是通过`模板的特化`，提供work_type。



例如：




```Python
class Staff
{
};

template <>
class TypeTraits<Staff>
{
public:
    typedef brain_worker worker_type;
};
```
		

测试仍然正常：




```C++
Staff s;
distinction(s);
```
		



 


 



###进一步简化


 


这里我们考虑的是内置的情形。对于那些要内置type的类，如果type个数过多，程序编写就容易出现问题，我们考虑使用继承，先定义一个base类：




```Python
template <typename T>
struct type_base
{
    typedef T worker_type;
};
```
		

所有的类型，`通过public继承这个类即可`：




```C++
class Worker : public type_base<physical_worker>
{
};

class Farmer : public type_base<physical_worker>
{
};

class Teacher : public type_base<brain_worker>
{
};

class Doctor : public type_base<brain_worker>
{
};
```
		

 


看到这里，我们应该明白，traits相对于简单内置类型的做法，`强大之处在于：如果一个类型无法内置type，那么就可以借助函数特化，从而借助于traits`。而内置类型仅仅使用于class类型。


 


以STL中的迭代器为例，很多情况下我们需要辨别迭代器的类型，


例如distance函数计算两个迭代器的距离，有的迭代器具有随机访问能力，如vector，有的则不能，如list，我们计算两个迭代器的距离，就需要先判断迭代器能否相减，因为只有具备随机访问能力的迭代器才具有这个能力。


我们可以使用内置类型来解决。


可是，许多迭代器是使用指针实现的，指针不是class，无法内置类型，于是，STL采用了traits来辨别迭代器的类型。


 


最后，我们应该认识到，`traits的基石是模板特化`。


 


下篇文章，我们使用traits，来辨别一个类型是否是pod类型。

			