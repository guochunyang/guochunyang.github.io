---
layout: post
title: 从range和xrange的性能对比到yield关键字（上）
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---

###使用xrange
   
  当我们获取某个数量的循环时，我们惯用的手法是for循环和range函数，例如：
  

```C++
for i in range(10):
    print i
```
		

这里range（10）生成了一个长度为10的列表，内容为从0到9，所以这里的for循环实际上是在遍历其中的元素。


如果循环次数过大的时候，range要生成一个巨大的列表，这将导致程序的性能降低。


解决方案是采用xrange，用法基本与range相同：




```C++
for i in xrange(10):
    print i
```
		

但是二者的性能差距到底有多大？


 



###性能测评


 


我们使用下面的程序做一个测试：




```Python
from time import time
from time import sleep
import sys

def count_time():
    def tmp(func):
        def wrapped(*args, **kargs):
            begin_time = time()
            result = func(*args, **kargs)
            end_time = time()
            cost_time = end_time - begin_time
            print '%s called cost time : %s ms' %(func.__name__, float(cost_time)*1000)
            return result
        return wrapped
    return tmp

@count_time()
def test1(length):
    for i in range(length):
        pass

@count_time()
def test2(length):
    for i in xrange(length):
        pass

if __name__ == '__main__':
    length = int(sys.argv[1])
    test1(length)
    test2(length)
```
		

上面的代码中，count_time是一个装饰器，用于统计程序运行的时间。


我们下面开始正式的测试：




```C++
wing@ubuntu:~/Documents/py|⇒  python 10.py 100000
test1 called cost time : 13.8590335846 ms
test2 called cost time : 3.76796722412 ms
wing@ubuntu:~/Documents/py|⇒  python 10.py 100000
test1 called cost time : 16.725063324 ms
test2 called cost time : 3.08418273926 ms
wing@ubuntu:~/Documents/py|⇒  python 10.py 200000
test1 called cost time : 34.875869751 ms
test2 called cost time : 7.85899162292 ms
wing@ubuntu:~/Documents/py|⇒  python 10.py 500000
test1 called cost time : 41.6638851166 ms
test2 called cost time : 17.1940326691 ms
wing@ubuntu:~/Documents/py|⇒  python 10.py 500000
test1 called cost time : 59.8731040955 ms
test2 called cost time : 14.0538215637 ms
wing@ubuntu:~/Documents/py|⇒  python 10.py 500000
test1 called cost time : 94.1109657288 ms
test2 called cost time : 8.5780620575 ms
wing@ubuntu:~/Documents/py|⇒  python 10.py 500000
test1 called cost time : 61.615228653 ms
test2 called cost time : 7.21502304077 ms
```
		

结果令我们大吃一惊，二者的差距非常明显，最高的时候差距了十几倍。


我们再选取几个较小的数据：




```C++
wing@ubuntu:~/Documents/py|⇒  python 10.py 10    
test1 called cost time : 0.00596046447754 ms
test2 called cost time : 0.0109672546387 ms
wing@ubuntu:~/Documents/py|⇒  python 10.py 20
test1 called cost time : 0.00619888305664 ms
test2 called cost time : 0.159025192261 ms
wing@ubuntu:~/Documents/py|⇒  python 10.py 50
test1 called cost time : 0.00786781311035 ms
test2 called cost time : 0.00405311584473 ms
wing@ubuntu:~/Documents/py|⇒  python 10.py 100
test1 called cost time : 0.00786781311035 ms
test2 called cost time : 0.00309944152832 ms
```
		

这次range的性能并不差，甚至开始还略显高。


我们可以得出结论，`当n较小时，我们使用range，但当i超过一定范围时，我们就必须考虑使用xrange了`。


但是，二者性能差距的原因在哪里？


我们下文分析。

			