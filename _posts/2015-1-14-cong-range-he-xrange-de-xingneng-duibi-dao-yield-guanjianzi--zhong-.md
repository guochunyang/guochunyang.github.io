---
layout: post
title: 从range和xrange的性能对比到yield关键字（中）
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
上节提出了range和xrange的效率问题，这节我们来探究其中的原因
   
  
###yield的使用
   
  我们看下面的程序：
  

```Python
#coding: utf-8

def test():
    print 4
    print 2
    print 5

if __name__ == '__main__':
    test()
```
		



这段代码的运行结果当然是没有任何疑问的。


但是如果我将代码修改一下：




```Python
#coding: utf-8

def test():
    yield 4
    yield 2
    yield 5

if __name__ == '__main__':
    print test()
```
		

运行结果有些奇怪：




```C++
<generator object test at 0xb71f1144>
```
		

我们尝试这样使用：




```C++
if __name__ == '__main__':
    for i in test():
        print i
```
		



结果却出人意料：




```C++
wing@ubuntu:~/Documents/py|⇒  python 17.py
4
2
5
```
		



这是什么原因呢？这里看起来，test()好像一个集合，里面存储了4，2，5，所以我们才能够依次遍历。


实际上，原因并非如此。


当一个函数中含有yield时，`这个函数就不再是一个普通的函数`，而是一个可迭代的对象（实际上叫做生成器，不过现在不必关心概念）。


同样，执行该函数时，不再是马上执行其中的语句，而是生成一个可迭代对象。当执行迭代的时候，才真正运行其中的代码。



###并不是从头开始执行，而是从上次yield退出的位置继续执行


尝试下面的代码：




```Python
#coding: utf-8

def test():
    yield 4
    yield 2
    yield 5

if __name__ == '__main__':
    t = test()
    it = iter(t)
    print it.next()
    print it.next()
    print it.next()
    print it.next()
```
		

运行结果为：




```C++
wing@ubuntu:~/Documents/py|⇒  python 17.py
4
2
5
Traceback (most recent call last):
  File "17.py", line 14, in <module>
    print it.next()
StopIteration
```
		

从这里的结果可以看出，test()语句没有执行代码段，而是生成了一个可以迭代的对象。


我们甚至可以得出结论，`每当执行一次next，就向后执行到下一个yield语句`，或者所有的语句执行完毕。


 



###range的实现


 


我们尝试实现range：




```Python
#coding: utf-8

def _range(value):
    i = 0
    result = []
    while i < value:
        result.append(i)
        i += 1
    return result

if __name__ == '__main__':
    for i in _range(4):
        print i
```
		

range的逻辑比较简单，就是生成一个列表。


 



###xrange的模拟实现


 


我们根据前面的结论，猜测xrange是一个含有yield的函数，于是：




```Python
#coding: utf-8

def _xrange(value):
    i = 0
    while i < value:
        yield i
        i += 1

if __name__ == '__main__':
    for i in _xrange(4):
        print i
```
		

运行一下，结果和我们预期一致。


当然，实际的xrange比我们这里编写的更加复杂，但是基本原理是一致的。


 



###为何xrange比range高效？


 


答案很明显了，range是一次性生成所有的数据，而xrange，内部使用了yield关键字，每次只运行其中一部分，这样从头到尾都没有占用大量的内存和时间。所以效率较高。


 


我们再次比较性能，这次比较的是我们自己编写的版本：




```Python
#coding: utf-8
import sys
from time import time

def _range(value):
    i = 0
    result = []
    while i < value:
        result.append(i)
        i += 1
    return result

def _xrange(value):
    i = 0
    while i < value:
        yield i
        i += 1

def count_time(func):
    def wrapped(*args, **kargs):
        begin_time = time()
        result = func(*args, **kargs)
        end_time = time()
        cost_time = end_time - begin_time
        print '%s called cost time : %s ms' %(func.__name__, float(cost_time)*1000)
        return result
    return wrapped

@count_time
def test1(length):
    for i in _range(length):
        pass

@count_time
def test2(length):
    for i in _xrange(length):
        pass

if __name__ == '__main__':
    length = int(sys.argv[1])
    test1(length)
    test2(length)
```
		

运行结果为：




```C++
wing@ubuntu:~/Documents/py|⇒  python 19.py 1000
test1 called cost time : 0.116109848022 ms
test2 called cost time : 0.0619888305664 ms
wing@ubuntu:~/Documents/py|⇒  python 19.py 10000
test1 called cost time : 2.39086151123 ms
test2 called cost time : 0.566959381104 ms
wing@ubuntu:~/Documents/py|⇒  python 19.py 100000
test1 called cost time : 15.5799388885 ms
test2 called cost time : 6.41298294067 ms
wing@ubuntu:~/Documents/py|⇒  python 19.py 1000000
test1 called cost time : 130.295038223 ms
test2 called cost time : 65.4468536377 ms
wing@ubuntu:~/Documents/py|⇒  python 19.py 10000000
test1 called cost time : 13238.3038998 ms
test2 called cost time : 652.212142944 ms
```
		











显然，使用yield的版本更加高效。


 


下文，我们探究生成器。

			