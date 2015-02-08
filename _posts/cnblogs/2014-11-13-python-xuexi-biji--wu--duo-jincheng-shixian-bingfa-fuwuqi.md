---
layout: post
title: Python学习笔记（五）多进程实现并发服务器
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
 
  每创建一个TCP连接，就创建一个进程。
  代码如下：
  

```Python
# coding: utf-8
import socket
import os
import sys
import signal
import errno
from time import ctime

def hanlde_sigchld(a, b):
    (pid, status) = os.wait()
    print 'Child %d Finish, status = %d' % (pid, status)

def handle_connection(client_socket, client_address):
    while True:
        data = client_socket.recv(1024)
        if not data:
            print 'disconnect', client_address
            client_socket.close()
            break;
        else:
            client_socket.send('[%s] %s' % (ctime(), data)) #回显消息


if __name__ == '__main__':
    signal.signal(signal.SIGCHLD, hanlde_sigchld) #安装SIGCHLD的处理函数

    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    listen_address = ('localhost', 9981)
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server_socket.bind(listen_address)
    server_socket.listen(10)

    while True:
        try:
            (client_socket, client_address) = server_socket.accept()
        except IOError, e:            
            if e.errno == errno.EINTR:
                continue #继续等待
            else:
                raise #将异常向外throw
        print 'Got connection from ', client_address
        pid = os.fork()
        if pid == 0:
            server_socket.close()
            handle_connection(client_socket, client_address)
            sys.exit(0)    #防止子进程中忘记exit
        elif pid > 0:
            client_socket.close() #必须关闭
```
		



这里有几点需要注意：


1.子进程中需要关闭server套接字，因为子进程只需要客户套接字即可。


2.父进程必须关闭客户套接字，`因为该socket是基于引用计数的，父进程不关闭，会导致该套接字永远不会真正的关闭`。


3.注意处理子进程的消亡，避免僵尸进程。这里`不能直接使用wait或者waitpid函数，因为该函数会使得进程阻塞`，这样不具备并发能力。

			