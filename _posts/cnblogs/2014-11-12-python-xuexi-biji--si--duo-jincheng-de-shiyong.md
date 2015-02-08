---
layout: post
title: Python学习笔记（四）多进程的使用
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
python中多进程与Linux 下的C基本相同。
   
  
###fork的基本使用
   
  先看最简单的例子：
  

```Python
# coding: utf-8
import os

def my_fork():
    pid = os.fork()
    if pid == 0:
        print 'this is child, pid = %d, parent id = %d' % (os.getpid(), os.getppid())
    elif pid > 0:
        print 'this is parent, pid = %d, child id = %d' % (os.getpid(), pid)
        os.waitpid(pid, 0) #等待子进程结束

if __name__ == '__main__':
    my_fork()
```
		



这个例子演示了fork的基本使用，还有就是我们最后使用了waitpid来回收子进程。


如果不知道具体的子进程号码，可以使用wait函数。


 



###管道pipe的使用


 


代码如下：




```Python
# coding: utf-8
import os
from time import sleep

def my_fork():
    r, w = os.pipe()
    pid = os.fork()
    if pid == 0:
        os.close(r) #关闭读端
        w = os.fdopen(w, "w")
        for i in range(10):
            w.write('%s\n' % (str(i+1))) #最后加上\n
            w.flush()  #这里记得刷新
            sleep(0.5)
        w.close()
    elif pid > 0:
        os.close(w) #关闭写端
        r = os.fdopen(r, "r")
        while True:
            data = r.readline() #不要使用read
            if not data:
                print 'close.'
                break;
            print 'received : %s' % (data)
        os.waitpid(pid, 0)  #等待子进程结束

if __name__ == '__main__':
    my_fork()
```
		



在子进程中，连续10次发送数字。


这里有几点值得注意：



  write时加上\n符号


  接收时使用readline函数


  每发送完一个数据，就刷新flush一次缓冲区



 



###使用信号处理僵尸进程


 


Python中也可以使用信号处理函数，例如最简单的中断信号：




```Python
# coding: utf-8
import os
import signal
from time import sleep

def handler(a, b):
    print 'Ctrl + C'

if __name__ == '__main__':
    signal.signal(signal.SIGINT, handler)
    while True:
        pass
```
		

每按一次Ctrl+C，就触发一次这个函数。


 


代码如下：




```Python
# coding: utf-8
import os
import signal
from time import sleep

def handler(a, b):
    (pid, status) = os.wait()
    print 'Child %d Finish, status = %d' % (pid, status)

def my_fork():
    pid = os.fork()
    if pid == 0:
        print 'this is child, pid = %d, parent id = %d' % (os.getpid(), os.getppid())
    elif pid > 0:
        print 'this is parent, pid = %d, child id = %d' % (os.getpid(), pid)
        
        while True:
            pass

if __name__ == '__main__':
    signal.signal(signal.SIGCHLD, handler)
    my_fork()
```
		


每当有子进程消亡，就触发SIGCHLD信号，然后在处理函数中调用wait函数。这里比Linux下简单，`不必使用while循环回收`。

 


下节使用python，编写一个多进程的并发服务器。


完。

			