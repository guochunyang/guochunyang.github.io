---
layout: post
title: C++11之右值引用（二）：右值引用与移动语义
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
上节我们提出了右值引用，可以用来区分右值，那么这有什么用处？
   
  
###问题来源
   
  我们先看一个C++中被人诟病已久的问题：
  我把某文件的内容读取到vector中，用函数如何封装？
  大部分人的做法是：
  

```C++
void readFile(const string &filename, vector<string> &words)
{
    words.clear();
    //read XXXXX
}
```
		



这种做法完全可行，但是代码风格谈不上美观，我们尝试着这样编写：




```C++
vector<string> readFile(const string &filename)
{
    vector<string> ret;
    ret.push("cesfwfgw");

    //....
    //

    return ret;
}
```
		



这样我们就可以在main中这样调用：




```C++
vector<string> coll = readFile("fef.text");
```
		



但是，稍微熟悉C++的都知道，这样在语法上会造成大量的开销：



  ret复制给临时变量，该临时变量开辟在heap上




  临时变量复制给coll



这中间产生两次复制和销毁的开销。


如果说这个例子，可以采用开头的代码解决开销，那么如果是一个查询返回结果的函数，那么我们必须这样写：




```C++
vector<string> queryWord(const string &word)
{
    vector<string> result;
    //XXXXX

    return result;
}
```
		



这里的开销就无法避免了。


 



###移动语义的引入


 


我们考虑一个生活中常见的问题（这里参考了<a href="http://www.zhihu.com/question/22111546">如何评价 C++11 的右值引用（Rvalue reference）特性？</a>），如果把一个很重的货物从箱子A移动到箱子B，那么



  正常的做法是：打开箱子A，把物品搬出来，移动到B，然后关上A。




  另一种比较奇葩的做法是：在B中复制一个物品A，然后将A中的销毁。




  更奇葩的做法是：由于复制工具的局限性，我们无法直接在B中复制，所以我们只好先在地上复制一个物品temp，销毁A中的物品，然后根据temp在B中再复制一份，再销毁地上的temp。



事实上，C++98采用的就是最后一种效率极其低下的做法，这里的`关键在于，C++98没有很好的区分“copy”和“move”语义`。


上述问题中，我们明确提出移动A到B中，但是C++98由于移动语义的缺失，只好采用copy的方式完成。


 


我们再回到开头的问题中：




```C++
vector<string> readFile(const string &filename)
{
    vector<string> ret;
    ret.push("cesfwfgw");

    //....
    //

    return ret;
}
```
		

这里我们必须看到一点，在完成函数调用后，ret就要被销毁，所以我们想到一个主意，不是把ret中的内容复制给main中的coll，而是`将ret中的内容偷到coll中`，然后将ret悄悄的销毁。


这样是可行的，因为ret的生命周期很短。


 



###哪些可以偷？


 


现在问题来了，C++中哪些可以偷，哪些不能？


我们回顾上一节提到的四个表达式：




```C++
string one("one");
const string two("two");
string three() { return "three"; }
const string four() { return "four"; }
```
		



显然，one和two生命周期较长，不能偷。four具有const属性，拒绝被偷。


那么three是可以被偷取的，因为它是临时变量，又没有const属性。


所以，`C++中的非const右值，和移动语义完全匹配`。



###上节我们提出用右值引用区分右值，正是为了解决哪些可以偷的问题！


 


OK，我们的思路已经很清晰了：



  1.为了解决返回对象开销问题，我们提出“偷取”，而不是复制对象




  2.我们面临哪些能偷，哪些不能偷的问题。




  3.右值可以偷取，所以我们如何区分右值？




  4.我们引入右值引用X &&来区分右值。



这就是右值引用的来源。


 


如果一个变量不是右值，但是我们又需要偷取，那么我们可以`采用std::move函数，将其强制转化为右值引用`。


例如：




```C++
void test(string name)
{
    string temp(std::move(name));
    // XXXXXX
}
```
		

注意，被偷取之后的name无法继续使用，所以move函数不可以随意使用。


 



###带来的影响


 


那么，右值引用带来哪些该变呢？


首先是类的成员函数赋值，看下面代码：




```C++
class People
{
public:
    People() 
    {
        cout << "People()" << endl;
    }
    People(const string &name)
    : name_(name)
    {
        cout << "People(const string &name)" << endl;
    }
    People(string &&name)
    : name_(name)
    {
        cout << "People(string &&name)" << endl;
    }

private:
    string name_;
};
```
		



这里name赋值，我们相对于C++98，提供了一个右值函数，将name的值移动给name_。


事实上，上面的两个函数可以合成一个：




```C++
class People
{
public:
    People() 
    {
        cout << "People()" << endl;
    }

    People(string name)
    : name_(std::move(name))
    {
        cout << "People(string name)" << endl;
    }

private:
    string name_;
};
```
		



这里注意，上面的name采用传值，并没有带来开销，因为：



  如果name传入的是一个右值，那么name本身采用移动构造，开销比复制小很多，相当于People(string &&name)


  <pre>`如果name传入的其他值，那么name是复制构造，然后移动给name_，也没有增加额外的开销。`</pre>


<pre>`对于构造函数，除了提供复制构造函数，还需要<font color="#ff0000">移动构造函数`。如下：</font></pre>



```C++
class People
{
public:
    People() 
    {
        cout << "People()" << endl;
    }

    People(string name)
    : name_(std::move(name))
    {
        cout << "People(string name)" << endl;
    }

    People(const People &p)
    : name_(p.name_)
    {
        cout << "People(const People &p)" << endl;
    }
    People(People &&p)
    : name_(std::move(p.name_))
    {
        cout << "People(People &&p)" << endl;
    }

private:
    string name_;
};
```
		



`注意在最后一个`People(People &&p)中，`移动p内的name时，必须显式使用move`，来强制移动name成员。


同样，还有`移动赋值运算符`。


``


`另外，在C++98中，容器内的元素必须具备值语义，现在则不同，元素具备移动能力即可，后文我们在智能指针系列会提到unique_ptr，它可以放入vector中，但是不具有复制和赋值能力。`


``


`其他的影响请参考：<a href="http://www.zhihu.com/question/22111546">如何评价 C++11 的右值引用（Rvalue reference）特性？</a>`


``


`下文通过一个string的模拟实现，演示右值引用的使用。`

			