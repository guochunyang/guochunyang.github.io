---
layout: post
title: Python学习笔记（七）函数的使用
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
python中的函数使用较简单，这里列出值得注意的几点：
   
  
###内嵌函数
   
  例如：
  

```Python
# coding: utf-8

def foo():
    def bar():
        print 'bar() called.'
    print 'foo() called.'

foo()
bar()
```
		

对bar的调用是非法的，因为bar的作用域仅限于foo内，除非使用闭包将其返回。




```Python
# coding: utf-8

def foo():
    def bar():
        print 'bar() called.'
    print 'foo() called.'
    return bar

s = foo()
s()
```
		

此时对s的调用就是合法的，因为我们在外部引用了bar。


 


 





###装饰器


 


我们在类中编写static函数时，就使用了static包装器，如下：




```Python
# coding: utf-8

class Test():
    @staticmethod
    def foo():
        print 'static method'

Test.foo()
```
		



实际上，`包装器就是函数`。


下面看一个demo：




```Python
# coding: utf-8

from time import ctime

def testfunc(func):
    def wrappedFunc():
        print '[%s] %s() called.' % (ctime(), func.__name__)
        return func()
    return wrappedFunc

@testfunc
def foo():
    print 'foo called.'


foo()
```
		



运行结果如下：




```C++
5:34:25 wing@ubuntu func python 2.py  
[Fri Nov 14 05:36:31 2014] foo() called.
foo called.
```
		

上面如果不对foo添加装饰器，还可以这样调用：




```Python
# coding: utf-8

from time import ctime

def testfunc(func):
    def wrappedFunc():
        print '[%s] %s() called.' % (ctime(), func.__name__)
        return func()
    return wrappedFunc

#@testfunc
def foo():
    print 'foo called.'

foo = testfunc(foo)
foo()
```
		
效果相同。

 


 



###函数的参数


 


在C++中，参数的位置是绝对不可以更改的，但是python中如果指定参数名，那么可以更改，例如：




```Python
# coding: utf-8

def foo(a, b):
    print 'a = %d, b = %d' % (a, b)

foo(4, 3)
foo(a = 4, b = 3)
foo(b = 3, a = 4)
```
		

最后一行调用，a和b就更换了位置。


 



###函数的缺省参数


 




```Python
# coding: utf-8

def foo(a, b = 20):
    print 'a = %d, b = %d' % (a, b)

foo(23)
foo(a = 12)
foo(4, 3)
foo(a = 4, b = 3)
foo(b = 3, a = 4)
```
		

存在缺省参数时，也可以指定参数名，这样就可以调换位置


再比如：










```Python
# coding: utf-8

def foo(name, age = 20, sex = 'male'):
    print 'name = %s, age = %d, sex = %s' % (name, age, sex)

foo('zhangsan', 23, 'male')
foo('lisi')
foo('lisi', sex = 'female')
foo(sex = 'male', name = 'gaga')
foo('haha', 34)
foo(age = 23, name = 'lucy')
foo(sex = 'female', age = 88, name = 'wangwu')
```
		

 



###不带关键字的可变参数


 


C语言中的printf函数，可以使用可变参数，意思就是参数的个数和类型是不确定的，Python同样支持这种用法。




```Python
# coding: utf-8

def foo(a, b = 'default Value', *theList):
    print 'a = ', a
    print 'b = ', b
    for i in theList:
        print 'other argument :', i

foo('abc')
foo(23, 4.56)
foo('abc', 123, 'xyz')
foo('abc', 123, 'xyz', 'haha', 'gaga', 34)
```
		

我们在参数的最后一个位置写入*theList，意思就是多余的参数写入一个元组中。


注意这里的参数都是不带关键字的，如果我们使用了c = 5，那么导致运行错误。


 



###带关键字的可变参数


 


如果我们真的需要使用c = 5这种额外的参数，可以使用**theRest，将多余的参数放入字典。




```Python
# coding: utf-8

def foo(a, b = 'default Value', **theDict):
    '除了a和b外，其余的参数放入'
    print 'a = ', a
    print 'b = ', b
    for eachKey in theDict.keys():
        print 'Other argument %s: %s' % (eachKey, theDict[eachKey])

foo('abc')
foo(23, 4.56)
foo('abc', 123, c = 'xyz')
foo('abc', 123, c = 'xyz', d = 'haha', e = 'gaga')
foo(c = 'xyz', a = 'haha', b = 'gaga')
foo('xyz', c = 'haha', b = 'gaga')
foo('hehe', c = 'c')
```
		

 


二者还可以结合使用：




```Python
# coding: utf-8

def foo(a, b = 'default Value', *theList, **theDict):
    print 'a = ', a
    print 'b = ', b
    for i in theList:
        print 'argument :', i
    for eachKey in theDict.keys():
        print 'Other argument %s: %s' % (eachKey, theDict[eachKey])

foo('abc', 123, 'zhangsan', c = 'xyz', d = 'haha', e = 'gaga')
```
		

 


对于上面的代码，尝试调用：




```C++
foo(2, 4, *(6, 8), **{'foo' : 12, 'bar' : 24})
```
		

运行结果为：



  

```C++
a =  2
b =  4
argument : 6
argument : 8
Other argument foo: 12
Other argument bar: 24
```
		
这与我们手工列出各项变量，结果是一致的。


注意不带关键字的集合未必是元组，只要是序列就可以，于是：




```Python
foo(2, 4, *"abcdefg", **{'foo' : 12, 'bar' : 24})
```
		

打印结果为：




```C++
a =  2
b =  4
argument : a
argument : b
argument : c
argument : d
argument : e
argument : f
argument : g
Other argument foo: 12
Other argument bar: 24
```
		

如果我们将元组和字典提前定义，如下：




```C++
aTuple = (6, 8)
bDict = {'foo' : 12, 'bar' : 24}
foo(2, 4, *aTuple, **bDict)
```
		

结果与上面是一致的。


不过值得注意的是：




```C++
foo(1, 2, 3, x = 4, y = 5, *aTuple, **bDict)
```
		

结果为：




```C++
a =  1
b =  2
argument : 3
argument : 6
argument : 8
Other argument y: 5
Other argument x: 4
Other argument foo: 12
Other argument bar: 24
```
		

解释器自动帮我们进行了合并。


 


这里我们应该认识到，`*``和**在函数调用时的作用就是讲元组或者字典展开，这与C语言中的*解引用操作语义一致`。

			