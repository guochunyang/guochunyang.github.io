---
layout: post
title: 一个用于统计函数调用消耗时间的装饰器
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
装饰器前面提过了，采用python的闭包特性实现：
  

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

@count_time()
def test():
    sleep(0.5)

if __name__ == '__main__':
    test()
```
		

我们尝试以下的代码：




```Python
class Test:
    @count_time()
    def test(self):
        print 'haha'

if __name__ == '__main__':
#    test()
    a = Test()
    a.test()
```
		

代码仍然可以正常工作，因为a.test()仅仅就是给test添加了一个额外的参数a而已。

			