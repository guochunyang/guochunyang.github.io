---
layout: post
title: Python闭包的高级应用-装饰器的实现
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
我们先看一个闭包的例子：
  

```Python
from time import ctime

def before_call(f):
    def wrapped(*args, **kargs):
        print 'before calling, now is %s' % ctime()
        return f(*args, **kargs)
    return wrapped

def test(name):
    print 'hello, %s' % (name)

if __name__ == '__main__':
    before_call(test)("lucky")
```
		



我们先看运行结果：




```C++
~/Documents/py python 2.py 
before calling, now is Sat Dec 27 21:30:18 2014
hello, lucky
```
		



上面的代码使用了闭包，因为子函数wrapped将父函数的内部变量f与之绑定。


这样，wrapped这个闭包函数，实际上先打印时间，然后调用f，所以正如结果打印的一般，`before_call起到的是一种装饰的作用`。


 


这里我扩展它的功能，增加一个调用函数后，打印时间：




```Python
from time import ctime

def before_call(f):
    def wrapped(*args, **kargs):
        print 'before calling, now is %s' % ctime()
        return f(*args, **kargs)
    return wrapped

def after_call(f):
    def wrapped(*args, **kargs):
        try:
            return f(*args, **kargs)
        finally:
            print 'after calling, now is %s' % ctime()
    return wrapped

def test(name):
    print 'hello, %s' % (name)

if __name__ == '__main__':
    before_call(test)("lucky")
    after_call(test)("peter")
    before_call(after_call(test))("john")
    after_call(before_call(test))('marry')
```
		



运行结果为：




```C++
~/Documents/py python 2.py 
before calling, now is Sat Dec 27 21:37:24 2014
hello, lucky
hello, peter
after calling, now is Sat Dec 27 21:37:24 2014
before calling, now is Sat Dec 27 21:37:24 2014
hello, john
after calling, now is Sat Dec 27 21:37:24 2014
before calling, now is Sat Dec 27 21:37:24 2014
hello, marry
after calling, now is Sat Dec 27 21:37:24 2014
```
		

运行结果是正确的。注意最后两个，顺序交换了，对结果无影响。


 


下面我们再包装一层：




```Python
def after_call():
    def after(f):
        def wrapped(*args, **kargs):
            try:
                return f(*args, **kargs)
            finally:
                print 'after calling, now is %s' % ctime()
        return wrapped
    return after


def before_call():
    def before(f):
        def wrapped(*args, **kargs):
            print 'before calling, now is %s' % ctime()
            return f(*args, **kargs)
        return wrapped
    return before
```
		



那么如何使用呢？这里就是python装饰器的语法，


如果我们这样使用：




```Python
@before_call()
def test(name):
    print 'hello, %s' % (name)

if __name__ == '__main__':
    test("lucky")
```
		



注意test函数前加了装饰的符号。


还可以这样：




```Python
@after_call()
def test(name):
    print 'hello, %s' % (name)
```
		

甚至可以嵌套多层：




```Python
@before_call()
@after_call()
def test(name):
    print 'hello, %s' % (name)
```
		



 


这就是python中装饰器的原理，内部采用了闭包。

			