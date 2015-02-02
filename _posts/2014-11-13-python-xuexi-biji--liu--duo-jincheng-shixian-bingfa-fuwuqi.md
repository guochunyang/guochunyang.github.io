---
layout: post
title: Python学习笔记（六）多进程实现并发服务器
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
这个相对于多进程更加简单，每accept一个新的连接就创建一个新的线程。代码如下：
  

```Python
# coding: utf-8
import socket
import sys
import errno
import threading
from time import ctime

class ClientThread(threading.Thread):
    def __init__(self, client_socket, client_address):
        threading.Thread.__init__(self)
        self.client_socket = client_socket
        self.client_address = client_address

    def run(self):
        self.handle_connection()

    def handle_connection(self):
        while True:
            data = self.client_socket.recv(1024)
            if not data:
                print 'disconnect', self.client_address
                self.client_socket.close()
                break;
            else:
                self.client_socket.send('[%s] %s' % (ctime(), data)) #回显消息


if __name__ == '__main__':
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
        t = ClientThread(client_socket, client_address)
        t.start()
```
		

注意这里`的thread不能进行join，否则会阻塞主线程，丧失并发能力`。


另外，`python中的线程不需要进行detach`。

			