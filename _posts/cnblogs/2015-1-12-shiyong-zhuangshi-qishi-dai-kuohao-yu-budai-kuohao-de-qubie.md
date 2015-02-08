---
layout: post
title: 使用装饰器时带括号与不带括号的区别
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
之前我们在[一个用于统计函数调用消耗时间的装饰器](http://www.cnblogs.com/inevermore/p/4194584.html)中写了一个装饰器，用于统计函数调用时间。代码如下：
  

```Python
from time import time
from time import sleep

def count_time():
    def tmp(func):
        def wrapped(*args, **kargs):
            begin_time = time()
            result = func(*args, **kargs)
            end_time = time()
            cost_time = end_time - begin_time
            print '%s called cost time : %s' %(func.__name__, cost_time)
            return result
        return wrapped
    return tmp
```
		

对于该装饰器，我们必须这样使用：




```Python
@count_time()
def test():
    sleep(0.5)

if __name__ == '__main__':
    test()
```
		

这里注意，装饰器后面加了括号。


如果我们不加括号：




```Python
@count_time
def test():
    sleep(0.5)

if __name__ == '__main__':
    test()
```
		

就会产生错误：




```C++
Traceback (most recent call last):
  File "16.py", line 16, in <module>
    @count_time
TypeError: count_time() takes no arguments (1 given)
```
		

 


但是很多装饰器使用时，是不必加括号的，那么这是怎么回事？


 


我们将上面的装饰器进行改写：




```Python
from time import time
from time import sleep
import sys

def count_time(func):
    def wrapped(*args, **kargs):
        begin_time = time()
        result = func(*args, **kargs)
        end_time = time()
        cost_time = end_time - begin_time
        print '%s called cost time : %s ms' %(func.__name__, float(cost_time)*1000)
        return result
    return wrapped
```
		

此时，就不需要加括号了。


这二者的区别在于，`第一个存在括号，允许用户传入自定义信息，所以需要额外包装一层`，不加括号的版本则不需要。


 


所以当我们需要自定义装饰器内的某些message时，就需要采用加括号的方式。


对于这个统计时间的装饰器，我们可以这样自定义信息：




```Python
#coding: utf-8
from time import time
from time import sleep

def count_time(msg):
    def tmp(func):
        def wrapped(*args, **kargs):
            begin_time = time()
            result = func(*args, **kargs)
            end_time = time()
            cost_time = end_time - begin_time
            print 'msg: %s ,%s called cost time : %s' %(msg, func.__name__, cost_time)
            return result
        return wrapped
    return tmp
```
		

然后这样使用：




```Python
@count_time("foobar")
def test():
    sleep(0.5)

@count_time("测试消息")
def test2():
    sleep(0.7)

if __name__ == '__main__':
    test()
    test2()
```
		

结果如下：




```C++
msg: foobar ,test called cost time : 0.501540899277
msg: 测试消息 ,test2 called cost time : 0.701622009277
```
		

 


后面综合前面几篇，写一个完整的装饰器教程。

			