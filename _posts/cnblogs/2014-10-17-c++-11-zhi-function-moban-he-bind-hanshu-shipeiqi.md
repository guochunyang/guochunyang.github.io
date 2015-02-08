---
layout: post
title: C++11之function模板和bind函数适配器
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
在C++98中，可以使用函数指针，调用函数，可以参考之前的一篇文章：[类的成员函数指针和mem_fun适配器的用法](http://www.cnblogs.com/inevermore/p/4014429.html)。
   
  
###简单的函数调用
   
  对于函数：
  

```C++
void foo(const string &s)
{
    cout << s << endl;
}
```
		

可以使用：




```C++
void (*pFunc) (const string &) = &foo;
pFunc("bar");
```
		

现在，我们使用C++的fumction，这个函数的返回值为void，参数为const string &，所以function的模板参数为void (const string&)，如下：




```C++
function<void (const string&)> f = &foo;
 f("bar");
```
		

再看另外一个例子：




```C++
void foo(int i, double d)
{
    cout << i << d << endl;
}


int main(int argc, const char *argv[])
{
    function<void (int, double)> f = &foo;
    f(12, 4.5);
    

    return 0;
}
```
		







 


 



###类的成员函数


 


下列代码：




```C++
class Foo
{
    public:
        void foo(int i) { cout << i << endl; }        

        static void bar(double d) { cout << d << endl; }

};
```
		

我们知道class的普通成员函数，含有一个隐式参数，而static则不然。


对于foo函数，可以使用C++98使用的mem_fun：




```C++
Foo f;
(mem_fun(&Foo::foo))(&f, 123);
```
		



现在我们使用function，但是隐式参数如何解决？我们使用bind，这是一种非常强大的函数适配器。使用方法如下：




```C++
function<void (int)> pf = bind(&Foo::foo, 
                               &f, 
                               std::placeholders::_1);
pf(345);
```
		

std::placeholders::_1叫做占位符，`如果使用bind绑定某个值，那么该函数等于该参数消失了。如果需要保留，需要使用占位符占住位置`。


在这个例子中，foo原本有两个参数，我们把隐式参数绑定某个对象地址，所以只剩下一个地址。function的模板参数为void ()。


我们也可以将两个全部占住，这样bind`和mem_fun的效果是一样的`：




```C++
function<void (Foo*, int)> pf2 = bind(&Foo::foo,
                                          std::placeholders::_1,
                                          std::placeholders::_2);

pf2(&f, 456);
```
		

这里注意`，_1和_2指的是实际调用的实参位置`，所以我们可以通过调换_1和_2的位置实现将int和Foo*两个参数调换。




```C++
function<void (int, Foo*)> pf3 = bind(&Foo::foo,
                    std::placeholders::_2,
                    std::placeholders::_1);

pf3(567, &f);
```
		



 



###bind的灵活使用


 


下面通过一个更具体的例子，演示bind的用法。




```C++
void test(int i, double d, const string &s)
{
    cout << "i = " << i << " d = " << d << " s = " << s << endl;
}
```
		

基本使用很简单：




```C++
function<void (int, double, const string&)> f1 = &test;
 f1(12, 3.14, "foo");
```
		

1.现在将其转化为function<void (int, double)>类型。


我们分析一下：



  string类型的参数消失了，所以我们需要bind一个值


  其他int和double位置不变，使用_1和_2占住即可。



我们先使用
  

```C++
using namespace std::placeholders;
```
		
方便占位符的使用。代码如下：




```C++
function<void (int, double)> f2 = 
            std::bind(&test,
                      _1,
                      _2,
                      "foo");
```
		

2.转化为function<void (double, int, const string &)>


分析：



  参数仍为3个，不需要绑定任何值。


  int和double的顺序变了，所以_1为double，_2为int，`注意占位符指的是实参位置`。



代码：




```C++
function<void (double, int, const string &)> f3 = 
        std::bind(&test,
                  _2,
                  _1,
                  _3);
```
		

其他例子如下：




```C++
//3.void (*)(const string &, int)
    function<void (const string &, int)> f4 = 
        std::bind(&test,
                  _2,
                  3.4,
                  _1);


    //4. void (*) (const string &, int, double)
    function<void (const string&, int, double)> f5
        = std::bind(&test,
                    _2,
                    _3,
                    _1);
    
    //5. void (*)(int)
    function<void (int)> f6 = 
        bind(&test,
             _1,
             3.4,
             "bar");

    //6 void(*)(const string &)
    function<void (const string &)> f7 =
        bind(&test,
             12,
             4.5,
             _1);

    //7. void (*)()
    function<void()> f8 = 
        bind(&test,
             12,
             4.5,
             "bar");
```
		



 


完毕。

			