---
layout: post
title: C++11之右值引用（一）：从左值右值到右值引用
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
C++98中规定了左值和右值的概念，但是一般程序员不需要理解的过于深入，因为对于C++98，左值和右值的划分一般用处不大，但是到了C++11，它的重要性开始显现出来。
  C++98标准明确规定：
     左值是可以取得内存地址的变量。
      非左值即为右值。
   从这里可以看出，`可以执行&取地址的就是左值，其他的就是右值`。
  这里需要明确一点，`能否被赋值不是区分C++左值和右值的区别`。
  我们给出四个表达式：
  

```C++
string one("one");
const string two("two");
string three() { return "three"; }
const string four() { return "four"; }
```
		

这里四个变量表达式，后面两个是临时变量，不可以取地址，所以属于右值，前面两个就是左值。


这里写出一个函数：




```C++
void test(const string &s)
{
    cout << "test(const string &s):" << s << endl;
}
```
		

然后进行测试：




```C++
test(one);
test(two);
test(three());
test(four());
```
		

编译我们发现，这个test可以接受所有的变量。


我们使用另一个函数做测试：




```C++
void test(string &s)
{
    cout << "test(string &s):" << s << endl;
}
```
		

然后测试发现，`只有one可以通过调用`。


然后我们同时提供两个test函数，然后我们编译运行发现：




```C++
test(string &s):one
test(const string &s):two
test(const string &s):three
test(const string &s):four
```
		

 


所以我们可以得出结论：



  在C++中，const X&可以接受所有的变量


  X &只可以接受普通变量



同时我们可以看出C++重载决议的一个特点：



  当一个参数可以匹配多个函数时，总是匹配最精准的一个。



例如上面的one，它也可以接受const X&，但是当X&形式的参数存在时，立刻选择后者。显然后者是专门为其准备的，二者的语义最为符合。`X&包含有一种修改语义，这与one是一致的`。


 



###引入const属性


 


上面的四个表达式，我们只讨论了左值和右值，我们再加上const进行讨论。


所以：



  <pre>string one("one"); 属于非const左值
<a name="cl-7"></a>const string two("two");   const左值
<a name="cl-8"></a>string three() { return "three"; } 非const右值
const string four() { return "four"; } const右值</pre>


<pre>`左值右值的属性与const是正交的。`</pre>

<pre>`现在引入一个问题，<font color="#ff0000">如果有时候需要区分四种变量`，那么该使用什么方法？</font></pre>

<pre>`前面的讨论，我们知道X&可以用来区分one，但是剩下的三个都可以被const X&吞掉，显然我们需要为一些变量提供一些定制版本的参数，来让不同的变量优先选择不同的参数。`</pre>

<pre>`C++11提供了右值引用来专门区分右值，我们同时提供四个函数：`</pre>



```C++
void test(const string &s)
{
    cout << "test(const string &s):" << s << endl;
}

void test(string &s)
{
    cout << "test(string &s):" << s << endl;
}

void test(string &&s)
{
    cout << "test(string &&s):" << s << endl;
}

void test(const string &&s)
{
    cout << "test(const string &&s):" << s << endl;
}
```
		

我们使用C++11进行编译，发现：




```C++
test(string &s):one
test(const string &s):two
test(string &&s):three
test(const string &&s):four
```
		

我们得出`最佳匹配`：


 



  X & 匹配 非const左值


  const X& 匹配 const左值


  X && 匹配 非const右值


  const X && 匹配 const右值



然后，我们可以采用逐个函数进行测试，发现：



  X& 仅仅匹配 非const左值，这与之前的结论一致


  const X& 可以匹配所有的表达式


  X && 只可以匹配 非const右值


  const X &&可以匹配const和非const 右值



OK，我们的问题解决，当我们需要区分右值的时候，就可以使用右值引用。


事实上，我们一般不需要const X &&，所以我们使用下列三种参数：



   


  void test(string &s); `修改语义`




  void test(string &&s); `移动语义`，后文介绍




  void test(const string &s); `常量语义`



这三种语义已经足够，在C++98中我们只使用修改和常量语义，`在C++11中，正是为了实现移动语义，我们才需要区分右值，才需要引入右值引用`。


 


下文讲C++11右值引用实现移动语义。

			