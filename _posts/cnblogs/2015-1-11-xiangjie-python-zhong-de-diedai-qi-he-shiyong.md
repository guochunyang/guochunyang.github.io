---
layout: post
title: 详解Python中的迭代器和使用
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
对于一个列表，a = [1, 2, 3, 4]，我们最常见的遍历方式就是：
  

```C++
a = [1, 2, 3, 4]
for item in a:
    print item
```
		

这里我们研究一种新的方式，就是迭代器。


在C++的STL中大量使用了迭代器，迭代器的作用当然就是遍历容器中的元素，而且他的好处就在于分离了容器的实现和遍历操作，不管我们使用什么类型的容器，使用迭代器的操作几乎是如出一辙。


 


看下面的代码：




```C++
>>> a = [2, 3, 4]
>>> it = iter(a)
>>> print it.next()
2
>>> print it.next()
3
>>> print it.next()
4
>>> print it.next()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
>>>
```
		

在上面的代码中，iter函数创建了一个可以迭代的对象，然后每次调用next方法，都能从其中取出元素。当没有元素可以迭代时，便抛出一个异常StopIteration。


 


所以我们上面的for循环可以这样改写：




```C++
a = [1, 2, 3, 4]

it = iter(a)
item = None
while True:
    try:
        item = it.next()
    except StopIteration:
        break
    print item    #do_something
```
		

 



###如何创建迭代器


 


现在我们想对我们自定义的class进行迭代操作，应该怎么办？


这里的关键是实现__iter__和next两个函数。




```Python
#!/usr/bin/env python
#coding: utf-8

class IterList:
    def __init__(self, elem):
        self.iter = iter(elem) 
    def __iter__(self):
        return self
    def next(self):
        return self.iter.next()

if __name__ == '__main__':
    a = [1, 2, 3, 4]
    test = IterList(a)
    for item in test:
        print item
```
		

这里我们仅仅是对class内部持有的元素做了一个包装，我们的__iter__返回的是自身，next则是调用的存储的iter的next方法。


 


现在我们提供一个稍微复杂的版本，这个版本可以允许向next函数传递参数，指定取出几个值。




```Python
#!/usr/bin/env python
#coding: utf-8

class IterList:
    def __init__(self, elem):
        self.iter = iter(elem) 
    def __iter__(self):
        return self
    def next(self, howmany=1):
        result = []
        for i in range(howmany):
            try:
                result.append(self.iter.next())
            except StopIteration:
                raise
        return result

if __name__ == '__main__':
    s = range(20)
    test = IterList(s)
    print test.next()
    print test.next()
    print test.next(3)
```
		
这个例子能够让我们更加清晰的认识到next函数的工作原理。 

 



###用迭代器实现斐波那契数列


 


我们再给出最后一个关于斐波那契数列的例子：


对于斐波那契数列，我们可以这样实现：




```Python
def fab(max): 
    n, a, b = 0, 0, 1 
    L = [] 
    while n < max: 
        L.append(b) 
        a, b = b, a + b 
        n = n + 1 
    return L

if __name__ == '__main__':
    for i in fab(5):
        print i
```
		

上面的fab函数返回一个列表，记录斐波那契数列的值。


但是，当max过大的时候，fab就必须生成一个巨大的列表，这不仅占用大量内存，也会消耗过多的时间。


下面我们使用迭代器，给出一个更加高效的实现：




```Python
class Fab(object): 

    def __init__(self, max): 
        self.max = max 
        self.n, self.a, self.b = 0, 0, 1 

    def __iter__(self): 
        return self 

    def next(self): 
        if self.n < self.max: 
            r = self.b 
            self.a, self.b = self.b, self.a + self.b 
            self.n = self.n + 1 
            return r 
        raise StopIteration()

if __name__ == '__main__':
    for i in Fab(5):
        print i
```
		

这个版本高效在何处？


之前的版本是预先把一个巨大的结果生成，然后逐个去遍历，而这里调用Fab时，仅仅做了一个简单的初始化工作，`真正的计算则是发生在每次迭代调用next的时候`。所以这里不会占用过大的内存，而且不需要预先计算，也节约了时间。


 


本文最后部分参考了：[http://www.ibm.com/developerworks/cn/opensource/os-cn-python-yield/](http://www.ibm.com/developerworks/cn/opensource/os-cn-python-yield/)

			