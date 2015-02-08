---
layout: post
title: Python之闭包
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
最近要使用python，研究下闭包特性。
   
  看下列的代码：
  

```Python
def counter(start_at = 0):
    count = [start_at]
    def incr():
        count[0] += 1
        return count[0]
    return incr

if __name__ == '__main__':
    count = counter(5)
    print count()
    print count()

    count2 = counter(100)
    print count2()
    print count()
```
		

 


counter返回一个内部的函数，如果是别的语言，每次函数执行的结果应该是相同的，但是这里的运行结果是：




```C++
~/Documents/py python 1.py
6
7
101
8
```
		

显然，count和count2是两个相互独立的作用域。


 


python的作用域有两种，一是全局作用域，一个是局部作用域。


但是下面的代码：




```Python
def counter(start_at = 0):
    count = [start_at]
    def incr():
        count[0] += 1
        return count[0]
    return incr
```
		



如果只看第一行，那么count这个变量，生成于counter被调用，当函数调用结束时，count便销毁。


但是这里借助嵌套的函数incr，`延长了count的声明周期`。


这就形成了一种单独的作用域。


在上面的代码中，`count和count2内部持有了一个count变量作为它的状态`，所以count每次调用的结果是不同的。


当我们调用一次counter函数时，就产生了一个闭包函数，例如count，他的内部持有一个状态count。


所以上面的四次调用，产生了两个闭包函数。


当count和count2销毁时，内部持有的闭包变量才会销毁。

			