---
layout: post
title: 标准库Allocator的使用（一）
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
上一篇我们提到了new运算符以及它的工作步骤，其实无非是把两项工作独立出来：


1.申请原始内存

2.执行构造函数


delete也涉及了两个工作：


1.执行析构函数

2.释放原始内存


其实标准库提供了另外一种更加高级的手段实现内存的分配和构造，就是std::allocator<T>的职责。

 

allocator提供了四个操作：


a.allocate(num) 为num个元素分配内存

b.construct(p) 将p所指的元素初始化

destroy(p) 销毁p指向的元素

deallocate(p, num) 回收p指向的&ldquo;可容纳num个元素&rdquo;的内存空间


使用示例如下：



```C++
#include <iostream>
#include <string>
#include <vector>
#include <memory>
using namespace std;

class Test
{
    public:
        Test() { cout << "Test" << endl; }
        ~Test() { cout << "Test ..." << endl; }

        Test(const Test &t)
        {
            cout << "Copy ....." << endl; 
        }

    private:
        //Test(const Test &);
        //void operator(const Test &);
};


int main(int argc, const char *argv[])
{
    allocator<Test> alloc;
    Test *pt = alloc.allocate(3); //申请三个单位的Test内存
    //此时pt指向的是原始内存
    {
        alloc.construct(pt, Test()); //构建一个对象，使用默认值
        //调用的是拷贝构造函数
        alloc.construct(pt+1, Test());
        alloc.construct(pt+2, Test());
    }
    alloc.destroy(pt);
    alloc.destroy(pt+1);
    alloc.destroy(pt+2);

    alloc.deallocate(pt, 3);
    return 0;
}
```
		
这里注意，allocator提供的allocate函数与operator new函数区别在于返回值，所以前者更加安全。

还有一点，construct一次只能构造一个对象，而且调用的是拷贝构造函数。

标准库提供了三个算法用于批量构造对象(前提是已经分配内存)


<pre>uninitialized_fill(beg, end, val)   //以val初始化[beg,end)</pre>
<pre>uninitialized_fill_n(beg, num, val) //以val初始化beg开始的num个元素</pre>
<pre>uninitialized_copy(beg, end, mem) //以[beg, end)的各个元素初始化mem开始的各个元素</pre>

以上三个函数操控的对象都是原始内存示例如下：



```C++
#include <iostream>
#include <string>
#include <vector>
#include <memory>
#include <stdlib.h>
using namespace std;

class Test
{
    public:
        Test(int val) :val_(val) { cout << "Test" << endl; }

        ~Test() { cout << "Test ..." << endl; }

        Test(const Test &t)
            :val_(t.val_)
        {
            cout << "Copy ....." << endl; 
        }

        int val_;
};


int main(int argc, const char *argv[])
{

    Test *pt = (Test *)malloc(3 * sizeof (Test));

    Test t(12);

    uninitialized_fill(pt, pt + 3, t);
    cout << pt[0].val_ << endl;
    

    Test *pt2 = (Test *)malloc(2 * sizeof (Test));
    uninitialized_copy(pt, pt + 2, pt2);

    free(pt);
    free(pt2);</pre>
<pre>return 0;
}
```
		
这里注意标准库的copy、fill函数与uninitialized_系列函数的区别：


copy、fill等操作的是已经初始化对象的内存，因此调用的是赋值运算符

而uninitialized_针对的是原始内存，调用的是拷贝构造函数


OK，我们到此，可以总结出分配原始内存的三种手段：


1.使用malloc

2.使用operator new

3.allocator的allocate函数


这三者从上到下，是一个由低级到高级的过程。

那么执行构造函数，有两种手段：


1.使用placement new运算符

2.使用allocator的construct函数


 

最后，C语言中的数据都是POD类型，使用原始内存即可，但是C++中的大部分都是非POD类型，需要执行相应的初始化函数，所以，在C++中应该尽可能避免使用memcpy之类的直接操控原始内存的函数。

			