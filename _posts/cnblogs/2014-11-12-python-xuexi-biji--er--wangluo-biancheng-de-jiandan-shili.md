---
layout: post
title: Python学习笔记（二）网络编程的简单示例
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
Python中的网络编程比C语言中要简洁很多，毕竟封装了大量的细节。
  所以这里不再介绍网络编程的基本知识。而且我认为，从Python学习网络编程不是一个明智的选择。
   
  
###简单的TCP连接
  
###
  服务器代码如下：
  

```Python
import socket
from time import ctime

sock = socket.socket()
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
sock.bind(('localhost', 9981))
sock.listen(5)

while True:
    print 'waiting for connection ...'
    peersock, peeraddr = sock.accept()
    print '....connected from:', peeraddr

    while True:
        data = peersock.recv(1024)
        if not data:
            break
        peersock.send('[%s] %s' % (ctime(), data))

    peersock.close()
sock.close()
```
		



注意这里设置了地址复用。


这是一个时间戳服务器，同时server还将用户的输入直接回显过去。


 


客户端的代码如下：




```Python
import socket

sock = socket.socket()
sock.connect(('localhost', 9981))

while True:
    data = raw_input('> ')
    if not data:
        break;
    sock.send(data)
    data = sock.recv(1024)
    if not data:
        break
    print data

sock.close()
```
		



运行两边的代码，这里贴出客户端的运行结果：




```C++
22:56:08 wing@ubuntu python python 2.py                                                1 ↵
> foo
[Tue Nov 11 22:56:10 2014] foo
> bar
[Tue Nov 11 22:56:12 2014] bar
>
```
		

 


 



###简单的UDP连接


 


服务器代码如下：




```Python
from socket import *
from time import ctime

sock = socket(AF_INET, SOCK_DGRAM)
sock.bind(('localhost', 9981))

while True:
    print 'waiting for message ...'
    data, addr = sock.recvfrom(1024)
    sock.sendto('[%s] %s' % (ctime(), data), addr)
    print '...received from and returned to:', addr

sock.close()
```
		



 


客户端代码如下：




```Python
from socket import *

addr = ('localhost', 9981)
sock = socket(AF_INET, SOCK_DGRAM)

while True:
    data = raw_input('> ')
    if not data:
        break;
    sock.sendto(data, addr)
    data, addr = sock.recvfrom(1024)
    if not data:
        break
    print data

sock.close()
```
		

 


 


Python中还提供了其他一系列的高级组件，例如TcpServer、ForkingTcpServer和ThreadingTCPServer等，后面会写一篇文章，总结各种网络编程的模型，到时候再去介绍。

			