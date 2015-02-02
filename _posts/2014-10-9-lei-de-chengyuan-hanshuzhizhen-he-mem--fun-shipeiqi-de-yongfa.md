---
layout: post
title: 类的成员函数指针和mem_fun适配器的用法
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
先来看一个最简单的函数：
  

```C++
void foo(int a)
{
    cout << a << endl;
}
```
		



它的函数指针类型为




```C++
void (*)(int);
```
		



我们可以这样使用：




```C++
void (*pFunc)(int) = &foo;
pFunc(123);
```
		



这就是函数指针的基本使用。


 


类的成员函数


 


那么，对于类的成员函数，函数指针有什么不同呢？


我们观察下面的类：




```C++
class Foo
{
public:

    //void (Foo::*)(int)
    void foo(int a)
    {
        cout << a << endl;
    }

    //void (*)(int)
    static void bar(int a)
    {
        cout << a << endl;
    }
};
```
		



我们尝试使用：




```C++
void (*pFunc)(int) = &Foo::foo;
```
		



得到编译错误：



  error: cannot convert ‘void (Foo::*)(int)’ to ‘void (*)(int)’ in initialization



从上面的编译错误，我们可以得知，foo的函数指针类型绝对不是我们期望的void (*)(int)，而是void (Foo::*)(int)。


原因很简单，类的成员函数，含有一个隐式的参数this，所以foo实际是存在两个参数，Foo*和int。


那么我们尝试使用void (Foo::*)(int)类型，如下：




```C++
void (Foo::*pFunc2)(int) = &Foo::foo;
```
		



那么如何调用呢，我们采用下列两种方式：




```C++
Foo f;
(f.*pFunc2)(45678);
```
		

以及




```C++
Foo *pf = &f;
(pf->*pFunc2)(7865);
```
		

此时的使用方式是正确的。


 


那么bar函数是static函数，它具有什么特点呢？




```C++
void (*pFunc)(int) = &Foo::bar;
 pFunc(123);
```
		

我们发现，static函数和自由函数的指针类型一致。


 


既然foo含有一个隐式参数，那么能否将其转化出来呢？我们使用STL中的mem_fun，这是一种函数适配器。




```C++
Foo f;
//void (Foo::*)(int) -> void (*)(Foo*, int)
(mem_fun(&Foo::foo))(&f, 123);
```
		

我们可以看到，mem_func起到一种转化作用，将void (Foo::*)(int)类型的成员函数指针转化为void (*)(Foo*, int)，后者是一个自由函数类型的指针，可以自由调用。


 


完毕。

			