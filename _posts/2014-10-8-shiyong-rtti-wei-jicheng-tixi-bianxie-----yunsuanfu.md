---
layout: post
title: 使用RTTI为继承体系编写”==”运算符
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
`转载请注明出处：`<a title="http://www.cnblogs.com/inevermore/p/4012079.html" href="http://www.cnblogs.com/inevermore/p/4012079.html">`http://www.cnblogs.com/inevermore/p/4012079.html`</a>
   
  RTTI，指的是运行时类型识别技术。
   
  先看一个貌似无关的问题：
   
  
###为继承体系重载<<操作符
  
###
  有下面的一个继承体系，我们需要为其添加输出操作符，应该怎么办：
  

```C++
class Animal
{

};

class Cat : public Animal
{

};

class Dog : public Animal
{

};
```
		



我们的第一个办法就是为每个类添加operator<<函数，但是我们也可以尝试这样：




```C++
class Animal
{
public:
    virtual string toString() const = 0;
};

class Cat : public Animal
{
public:
    string toString() const
    {
        return "Cat";
    }
};

class Dog : public Animal
{
    string toString() const
    {
        return "Dog";
    }
};

ostream &operator<<(ostream &os, const Animal &a)
{
    return os << a.toString();
}
```
		



显然我们只提供了一个基类的输出运算符，但是因为const Animal &a的缘故，所有子类都可以放入。那么如果根据不同的类型打印不同的内容呢，我们使用了多态。


这样做的好处是什么？如果采用每个类都提供输出运算符的方案，那么我们必须牢记，每编写一个类就要为其添加一个操作符，这一点是很容易遗忘的，但是如果采用我们上面的代码，我们将toString写为虚函数，这样，`我们每次去重新实现toString就可以了`。虚函数被遗忘的可能性比<<操作符低很多。


 



###为继承体系重载==操作符


 


还是之前的类：




```C++
class Animal
{

};

class Cat : public Animal
{

};

class Dog : public Animal
{

};
```
		

这次我们需要为其提供==操作符，那么我们应该提供几个？为每个类提供一个显然是不够的，因为继承体系中，派生类对象可以被基类对象引用，所以`我们有时候需要将Animal &与Cat &进行比较`。




```C++
bool operator==(const Animal &, const Animal &);
bool operator==(const Cat &, const Cat &);
bool operator==(const Dog &, const Dog &);
bool operator==(const Animal &, const Cat &);
bool operator==(const Cat &, const Dog &);
//.............
```
		

看起来，五个还不够。就算只有五个，我们需要逐个实现也是非常恐怖。那么有没有好一些的解决方案呢？我们采用本文第一节的技巧。



  只重载一个Animal基类的==操作符




  每个类实现equal，这是一个虚函数



 




在继续编写代码之前，我们先搞清楚，两个相同的前提是，`他们的类型相同`，然后再去比较成员变量。


于是我们给出的解决方案如下：




```C++
#include <iostream>
#include <typeinfo>
#include <string>
using namespace std;

class Animal
{
public:
    virtual bool equal(const Animal &other) const = 0;
};

class Cat : public Animal
{
public:
    bool equal(const Animal &other) const
    {
        //Cat Animal
        if(const Cat *pc =  dynamic_cast<const Cat*>(&other)) 
        {
            return name_ == pc->name_;
        }
        return false;
    }

private:
    string name_;
};

class Dog : public Animal
{
public:
    bool equal(const Animal &other) const
    {
        //Dog Animal
        if(const Dog *pd = dynamic_cast<const Dog*>(&other))
        {
            return name_ == pd->name_;
        }
        return false;
    }
private:
    string name_;
};

bool operator==(const Animal &a, const Animal &b)
{
    return typeid(a) == typeid(b) && a.equal(b);
}

int main(int argc, char const *argv[])
{

    Cat c;
    Dog d;

    Animal *pa = &c;
    cout << (*pa == c) << endl;
    pa = &d;
    cout << (*pa == d) << endl;
    return 0;
}
```
		

我们先来看<<的重载：




```C++
bool operator==(const Animal &a, const Animal &b)
{
    return typeid(a) == typeid(b) && a.equal(b);
}
```
		



这里利用了&&的短路特性，一旦两个对象的类型不同，那么后面就没有必要比较。



  typeid是一个类型识别运算符，如果要识别的类型不是class或者不含有virtual函数，那么typeid指出静态类型。`如果class含有虚函数，那么typeid在运行期间识别类型。`



对equal的调用，显然使用了动态绑定，`总是根据对象的实际类型调用对应的equal版本`。


然后我们看一下Dog中equal的实现：




```C++
bool equal(const Animal &other) const
    {
        //Cat Animal
        if(const Cat *pc =  dynamic_cast<const Cat*>(&other)) 
        {
            return name_ == pc->name_;
        }
        return false;
    }
```
		

 


这里利用了dynamic_cast进行了“向下塑形”，dynamic_cast与static_cast有一些不同：



  static_cast发生在`编译期间`，如果转化不通过，那么编译错误，如果编译无问题，那么转化一定成功。`static_cast仍具有一定风险`，尤其是向下塑形时，将Base*转化为Derived*时，指针可以转化，但是指针未必指向Derived对象。




  dynamic_cast发生在`运行期间`，用于将Base的指针或者引用转化为派生类的指针或者引用，如果成功，返回正常的指针或引用，`如果失败，返回NULL（指针），或者抛出异常（bad_cast)`。



在<<的重载中，我们保证了equal两个参数的类型相同，那么我们为何还需要在equal中“向下塑形”呢？



  equal有可能被单独使用，所以other的类型未必和自己相同。




  如果不进行转换，other是无法访问name属性的，因为Animal中没有name。



记住下面的代码：




```C++
//pb是Base类型指针
    if(Derived *pd =  dynamic_cast<Derived*>(pb)) 
    {
        //转化成功
    }
    else
    {
        //失败
    }
```
		



这是一种使用dynamic_cast的`标准实践`。


这段代码的逻辑是非常严密的：



  
###pd的生存期只存在于转化成功的情况




  无法在dynamic_cast和测试代码之间插入代码



如果使用的是引用，可以这样：




```C++
try
    {
        const Derived &d = dynamic_cast<const Derived&>(b);
        //成功
    }
    catch(bad_cast)
    {
        //失败处理
    }
```
		



也可以将引用取地址，然后去转化指针。


 


typeid和dynamic_cast是实现RTTI的主要手段。


 


完毕。

			