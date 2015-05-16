---
layout: post
title: muduo库的简单使用-echo服务的编写
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
#muduo库的简单使用

muduo是一个基于事件驱动的非阻塞网络库，采用C++和Boost库编写。  
它的使用方法很简单，参考这篇文章：[TCP网络编程本质论](http://blog.csdn.net/Solstice/article/details/6171831#t0)

里面有这么几句：

	我认为，TCP 网络编程最本质的是处理三个半事件：

	连接的建立，包括服务端接受 (accept) 新连接和客户端成功发起 (connect) 连接。
	连接的断开，包括主动断开 (close 或 shutdown) 和被动断开 (read 返回 0)。
	消息到达，文件描述符可读。这是最为重要的一个事件，对它的处理方式决定了网络编程的风格（阻塞还是非阻塞，如何处理分包，应用层的缓冲如何设计等等）。
	消息发送完毕，这算半个。对于低流量的服务，可以不必关心这个事件；另外，这里“发送完毕”是指将数据写入操作系统的缓冲区，将由 TCP 协议栈负责数据的发送与重传，不代表对方已经收到数据。
	
所以，使用muduo库只需编写上面几处相关的逻辑即可。像套接字建立、epoll轮询这种例行公事的代码，我们不必再编写。

下面我们实现echo服务器，echo的核心逻辑只有一个，那就是将收到的信息回显给对方，所以这里我们只需关心消息到达这个事件即可。

代码如下：


```cpp

#include <muduo/base/Logging.h>
#include <muduo/base/Timestamp.h>
#include <muduo/net/TcpServer.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/TcpConnection.h>

using namespace muduo;
using namespace muduo::net;

void onConnection(const TcpConnectionPtr &conn)
{
    LOG_INFO << "EchoServer - " << conn->peerAddress().toIpPort() << " -> "
             << conn->localAddress().toIpPort() << " is "
             << (conn->connected() ? "UP" : "DOWN");
}

void onMessage(const TcpConnectionPtr &conn,
               Buffer *buf,
               Timestamp time)
{
    muduo::string msg(buf->retrieveAllAsString());
    LOG_INFO << conn->name() << " echo " << msg.size() << " bytes, "
                     << "data received at " << time.toString();
    conn->send(msg);
}

int main(int argc, const char *argv[])
{
    EventLoop loop;
    InetAddress addr("127.0.0.1", 8988);
    TcpServer server(&loop, addr, "echo");
    server.setConnectionCallback(&onConnection);
    server.setMessageCallback(&onMessage);
    server.start();
    loop.loop();
    return 0;
}
```

使用下列命令编译：

	g++ -o echo echo.cc -lmuduo_net -lmuduo_base -lpthread
	
客户端采用netcat即可：

	echo "hello" | nc localhost 8988
	
##基于对象的使用方法

上面的使用方式是采用了全局函数，我们还可以将echo服务器封装成一个类：


```cpp

#include <muduo/net/TcpServer.h>
#include <muduo/base/Logging.h>
#include <boost/bind.hpp>
#include <muduo/net/EventLoop.h>

class EchoServer
{
 public:
  EchoServer(muduo::net::EventLoop* loop,
             const muduo::net::InetAddress& listenAddr);

  void start();  // calls server_.start();

 private:
  void onConnection(const muduo::net::TcpConnectionPtr& conn);

  void onMessage(const muduo::net::TcpConnectionPtr& conn,
                 muduo::net::Buffer* buf,
                 muduo::Timestamp time);

  muduo::net::TcpServer server_;
};

EchoServer::EchoServer(muduo::net::EventLoop* loop,
                       const muduo::net::InetAddress& listenAddr)
  : server_(loop, listenAddr, "EchoServer")
{
  server_.setConnectionCallback(
      boost::bind(&EchoServer::onConnection, this, _1));
  server_.setMessageCallback(
      boost::bind(&EchoServer::onMessage, this, _1, _2, _3));
}

void EchoServer::start()
{
  server_.start();
}

void EchoServer::onConnection(const muduo::net::TcpConnectionPtr& conn)
{
  LOG_INFO << "EchoServer - " << conn->peerAddress().toIpPort() << " -> "
           << conn->localAddress().toIpPort() << " is "
           << (conn->connected() ? "UP" : "DOWN");
}

void EchoServer::onMessage(const muduo::net::TcpConnectionPtr& conn,
                           muduo::net::Buffer* buf,
                           muduo::Timestamp time)
{
  // 接收到所有的消息，然后回显
  muduo::string msg(buf->retrieveAllAsString());
  LOG_INFO << conn->name() << " echo " << msg.size() << " bytes, "
           << "data received at " << time.toString();
  conn->send(msg);
}


int main()
{
  LOG_INFO << "pid = " << getpid();
  muduo::net::EventLoop loop;
  muduo::net::InetAddress listenAddr(2007);
  EchoServer server(&loop, listenAddr);
  server.start();
  loop.loop();
}
```

后面陆续分析一些复杂的示例。
			