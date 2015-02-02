---
layout: post
title: Python学习笔记（一）类和继承的使用
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
一年前就打算学Python了，折腾来折腾去也一直没有用熟练，主要是类那一块不熟，昨天用Python写了几个网络编程的示例，感觉一下子迈进了很多。这几天把学习Python的笔记整理一下，内容尽量简洁。
   
   
  下面这个例子演示类的基本使用：
  

```Python
# coding:utf-8

class Test():
    s = '这是一个静态变量'
    def __init__(self):
        print '这里是构造函数'
        self.a = 1
        self.b = 12

    def __del__(self):
        print '这里是析构函数'

    def foo(self):
        print '普通成员函数'

    @staticmethod
    def bar():
        print '类的静态函数'


if __name__ == '__main__':
    t = Test()
    t.foo()
    Test.bar()
    print t.__class__
    print Test.__bases__
    print Test.s
```
		

很多书上很啰嗦的介绍Python的类，但是很多Python学习者本身已经具备了C++或者Java的基础，所以我直接尝试写了这个一个demo。


在Python中，构造函数使用了__init__，析构函数则使用了__del__。


在C++中，类的成员变量和函数都是编译之前就确定了，而Python可以再运行期确定。


上例中的s相当于静态变量，整个类共同拥有。


__init__函数中的self.a属于普通成员变量。如果你在某一个函数中使用了 




```C++
self.c = 'foo'
```
		
之类的语句，意味着你在这一行为该对象添加了一个数据成员。 

但是这里注意，`只有运行这一行之后，对象的数据成员才添加了c`。所以，`Python的成员变量是可以在运行过程中增减的`。


 


 


 


再看第二个示例，关于继承和组合的：




```Python
# coding:utf-8

class Base():
    def __init__(self, a, b):
        print 'Base construct.'
        self.a = a;
        self.b = b;
        self.t = Other()

    def __del__(self):
        print 'Base destroy.'

    def foo(self):
        print 'a = %s b = %s' % (self.a, self.b)

class Other():
    def __init__(self):
        print 'Other construct.'

    def __del__(self):
        print 'Other destroy.'

class Derived(Base):
    def __init__(self, a, b):
        Base.__init__(self, a, b)
        print 'Derived construct.'

    def __del__(self):
        Base.__del__(self)
        print 'Derived destroy.'

if __name__ == '__main__':
    d = Derived("foo", "bar")
    d.foo()
    print d.__class__
    print d.__class__.__bases__
    print Derived.__bases__
```
		

Base是基类，Derived从Base中继承，同时Other类的一个对象是Derived的一个数据成员。


在本例中，注意，Derived的构造函数中，必须手工调用Base的构造函数，析构函数也是相同的用法。


 


最后一个例子，关于基类和派生类的函数覆盖问题：




```Python
# coding:utf-8

class Foo():
    def test(self):
        print 'test in Base.'

class Bar(Foo):
    def test(self):
        print 'test in Derived.'

if __name__ == '__main__':
    b = Bar()
    b.test()
```
		

我们运行程序，打印的是：




```C++
test in Derived.
```
		
发现调用的是派生类的版本。这说明`派生类的test函数覆盖了基类的版本`。 

这里需要注意，与C++不同，这里的Bar中的test函数，即使改变了参数也无所谓，总之，只要函数与基类中的函数重名，那就构成了覆盖。


如果在Bar的test函数中想调用基类的版本，可以使用：




```C++
Foo.test(self)
```
		

完整的代码如下：




```Python
# coding:utf-8

class Foo():
    def test(self):
        print 'test in Base.'

class Bar(Foo):
    def test(self):
        Foo.test(self)
        print 'test in Derived.'

if __name__ == '__main__':
    b = Bar()
    b.test()
```
		

 


完。

			