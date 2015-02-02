---
layout: post
title: Python学习笔记（三）多线程的使用
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
这节记录学习多线程的心得。
   
  Python提供了thread模块，不过该模块的缺点很多，例如无法方便的等待线程结束，所以我们使用更加高级的threading模块。
   
  threading模块的使用一共三种模式：
     1.利用函数生成一个Thread实例
    2.利用函数生成一个可以调用的类对象，生成一个Thread实例
    3.从Thread派生一个子类，创建这个子类的实例
    
  
###利用函数生成Thread实例
   
  第一种使用方式最为简单，代码如下：
  

```Python
import threading
from time import sleep

def threadFunc():
    i = 10;
    while i > 0:
        print 'i = %d' % i
        i -= 1

if __name__ == '__main__':
    t = threading.Thread(target = threadFunc)
    t.start()
    t.join()
```
		



这段代码的逻辑很简单，就是在线程中执行threadFunc这个函数。


如果该函数需要参数的话，在
  

```C++
t = threading.Thread(target = threadFunc)
```
		
这一行添加一个参数即可，如下：




```Python
import threading
from time import sleep

def threadFunc(i):
    while i > 0:
        print 'i = %d' % i
        i -= 1

if __name__ == '__main__':
    t = threading.Thread(target = threadFunc, args = [10])
    t.start()
    t.join()
```
		

注意args参数必须使用元组或者列表。


 



###利用函数生成一个可以调用的类对象，生成一个Thread实例


 


我们先补充一些知识，C++中有函数对象，就是对某一个类重载函数调用操作符，那么该类的对象就可以当做函数来使用，python中也有同样的机制：




```Python
class Foo():
    def __call__(self):
        print 'foobar'

if __name__ == '__main__':
    f = Foo()
    f()
```
		

此例中f是一个对象，但可以当做函数使用。当调用f()时，解释器调用了Foo中的__call__方法，相当于C++中的operator()操作符被重载。


 


还有一个关于apply的知识点：




```Python
def test(i):
    print 'i = %d' % i 

if __name__ == '__main__':
    apply(test, [1])
```
		

apply可以这样调用函数。通过这种机制，`我们可以将函数存储起来，选择合适的时机注意调用`。




```Python
from random import randint

def foo(i):
    print 'i = %d' % i 
def bar(i):
    print 'i*i = %d' % (i*i)

class Foo():
    def __call__(self, i):
        print 'foobar: %d' % i

if __name__ == '__main__':
    funcs = [foo, bar, Foo()]
    for func in funcs:
        i = randint(1, 4)
        apply(func, [i])
```
		



 


于是我们可以将函数存储在类中，为该类提供__call__函数，此时这个类的对象也是可以执行的，所以我们利用这个对象去生成Thread。




```Python
#coding: utf-8
import threading
from time import sleep

class ThreadFunc(object):
    def __init__(self, func, args):
        self.func = func
        self.args = args

    def __call__(self):
        apply(self.func, self.args)

def loop(i):
    while i > 0:
        print 'i = %d' % i
        sleep(0.5)
        i -= 1

if __name__ == '__main__':
    print '在主线程内执行这个函数'
    t1 = ThreadFunc(loop, [5])
    t1()

    print '开始执行一个新的线程'
    t2 = threading.Thread(target = t1)
    t2.start()
    t2.join()
    print '线程执行完毕'

    print '开始执行一个新的线程'
    t3 = threading.Thread(target = ThreadFunc(loop, [3]))
    t3.start()
    t3.join()
    print '线程执行完毕'
```
		

t1是个ThreadFunc的实例，既可以直接执行，又可以使用它去生成Thread实例。


 



###从Thread派生一个子类，创建这个子类的实例


 


最简单的使用方式如下：




```Python
import threading
from time import sleep

class MyThread(threading.Thread):
    def __init__(self):
        threading.Thread.__init__(self)
        self.count = 10

    def run(self):
        while self.count > 0:
            print 'i = %d' % self.count
            sleep(1)
            self.count -= 1

if __name__ == '__main__':
    t = MyThread();
    t.start()
    t.join()
```
		

我们去继承Thread类，然后覆盖其中的run方法，这与Java的Thread使用相一致。


创建多个线程可以这样：




```Python
import threading
from time import sleep

class MyThread(threading.Thread):
    def __init__(self):
        threading.Thread.__init__(self)

    def run(self):
        print 'begin .....'
        sleep(5)
        print 'end.....'

if __name__ == '__main__':
    threads = []
    for i in range(10):
        t = MyThread()
        threads.append(t)
    for t in threads:
        t.start()
    for t in threads:
        t.join()
```
		





 


不过，目前我们的线程逻辑是固定的，可以借鉴第二种方式，从外部传入逻辑，存储起来。




```Python
#coding: utf-8
import threading
from time import sleep

class CustomThread(threading.Thread):
    def __init__(self, func, args):
        threading.Thread.__init__(self)
        self.func = func
        self.args = args

    def run(self):
        apply(self.func, self.args)

def loop(i):
    while i > 0:
        print 'i = %d' % i
        sleep(0.5)
        i -= 1

if __name__ == '__main__':
    t = CustomThread(loop, [10])
    t.start()
    t.join()
```
		



这里跟第二种不同的是：



  1.采用了继承，基类是Thread


  2.覆盖run方法，而不是提供__call__方法


  3.使用时直接创建该类的实例



以上三种，我个人感觉第三种最方便，在大一些程序中，可以将该Thread单独做成一个模块。


另外，`前两种的本质是一样的，都是向Thread传入一个可以执行的对象`（python中函数也是对象）。


 


完。

			