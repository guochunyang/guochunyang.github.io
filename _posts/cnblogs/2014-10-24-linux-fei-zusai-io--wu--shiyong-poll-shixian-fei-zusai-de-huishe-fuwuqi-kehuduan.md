---
layout: post
title: Linux非阻塞IO（五）使用poll实现非阻塞的回射服务器客户端
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
前面几节我们讨论了非阻塞IO的基本概念、Buffer的设计以及非阻塞connect的实现，现在我们使用它们来完成客户端的编写。
  我们在<a title="http://www.cnblogs.com/inevermore/p/4049165.html" href="http://www.cnblogs.com/inevermore/p/4049165.html">http://www.cnblogs.com/inevermore/p/4049165.html</a>中提出过，客户端需要监听stdin、stdout和sockfd。
  这里需要注意的是
     只有缓冲区可写的时候，才去监听sockfd和stdin的读事件。
    过去在阻塞IO中，我们`总是监听sockfd的读事件`，因为每当sockfd可读，我们就去调用用户的回调函数处理read事件，在回调函数中需要用户手工read缓冲区的数据。 换句话说，`接收数据是用户的责任，poll模型只需要提醒用户去接收即可`。
    而在非阻塞IO中，因为poll采用的是水平触发，如果缓冲区满了，每次read等于无效操作，那么数据始终`堆积在内核中`，poll会不停的被触发。`这在某种程度上等于轮询`。所以我们只在缓冲区可用的情况下监听sockfd的读事件。
      只有缓冲区可读的时候，才去监听sockfd和stdout的写事件。因为没有数据可写，监听write事件除了不停的触发poll之外，没有实际意义。
   所以每次执行poll之前，需要重新装填poll的events数组。
  完整的代码如下：
  

```C++
#include "sysutil.h"
#include "buffer.h"

int main(int argc, char const *argv[])
{
    //创建client套接字
    int sockfd = tcp_client(8934);
    //调用非阻塞connect函数
    int ret = nonblocking_connect(sockfd, "192.168.44.136", 9981, 5000);
    if(ret == -1)
    {
        fprintf(stderr, "Timeout .\n");
        exit(EXIT_FAILURE);
    }

    //将三个fd设置为Non-Blocking
    activate_nonblock(sockfd);
    activate_nonblock(STDIN_FILENO);
    activate_nonblock(STDOUT_FILENO);


    buffer_t recvbuf; //sockfd -> Buffer -> stdout
    buffer_t sendbuf; //stdin -> Buffer -> sockfd

    //初始化缓冲区
    buffer_init(&recvbuf);
    buffer_init(&sendbuf);

    struct pollfd pfd[10];

    while(1)
    {
        //初始化
        int ix;
        for(ix = 0; ix != 3; ++ix)
        {
            pfd[ix].fd = -1;
            pfd[ix].events = 0;
        }

        //重新装填events数组
        if(buffer_is_readable(&sendbuf))
        {
            pfd[0].fd = sockfd;
            pfd[0].events |= kWriteEvent;
        }
        if(buffer_is_writeable(&sendbuf))
        {
            pfd[1].fd = STDIN_FILENO;
            pfd[1].events |= kReadEvent;
        }
        if(buffer_is_readable(&recvbuf))
        {
            pfd[2].fd = STDOUT_FILENO;
            pfd[2].events |= kWriteEvent;
        }
        if(buffer_is_writeable(&recvbuf))
        {
            pfd[0].fd = sockfd;
            pfd[0].events |= kReadEvent;
        }

        //监听fd数组
        int nready = poll(pfd, 3, 5000);
        if(nready == -1)
            ERR_EXIT("poll");
        else if(nready == 0)
        {
            printf("timeout\n");
            continue;
        }
        else
        {
            int i;
            for(i = 0; i < 3; ++i)
            {
                int fd = pfd[i].fd;
                if(fd == sockfd && pfd[i].revents & kReadEvent)
                {
                    //从sockfd接收数据到recvbuf
                    if(buffer_read(&recvbuf, fd) == 0)
                    {
                        fprintf(stderr, "server close.\n");
                        exit(EXIT_SUCCESS);
                    } 
                }
                    
                if(fd == sockfd && pfd[i].revents & kWriteEvent)
                    buffer_write(&sendbuf, fd); //将sendbuf中的数据写入sockfd

                if(fd == STDIN_FILENO && pfd[i].revents & kReadEvent)
                {
                    //从stdin接收数据写入sendbuf
                    if(buffer_read(&sendbuf, fd) == 0)
                    {
                        fprintf(stderr, "exit.\n");
                        exit(EXIT_SUCCESS);
                    } 
                }

                if(fd == STDOUT_FILENO && pfd[i].revents & kWriteEvent)
                    buffer_write(&recvbuf, fd); //将recvbuf中的数据输出至stdout
            }
        }
    }

}
```
		

从以上的代码可以看出，大部分操作被封装进了buffer的实现中。


 


测试服务器，我暂时使用muduo库编写一个，代码如下：




```C++
#include <muduo/net/TcpServer.h>
#include <muduo/net/InetAddress.h>
#include <muduo/net/TcpConnection.h>
#include <muduo/base/Timestamp.h>
#include <muduo/net/EventLoop.h>
#include <muduo/base/Logging.h>
using namespace muduo;
using namespace muduo::net;

void onMessage(const TcpConnectionPtr &conn, Buffer *buf, Timestamp t)
{
    string s(buf->retrieveAllAsString());
    LOG_INFO << "recv msg : " << s.size() << " at: " << t.toFormattedString();
    conn->send(s);
}

int main(int argc, char const *argv[])
{
    EventLoop loop;
    InetAddress addr("192.168.44.136", 9981);
    TcpServer server(&loop, addr, "EchoServer");
    server.setMessageCallback(&onMessage);
    server.start();

    loop.loop();

    return 0;
}
```
		

读者如果使用上述的代码需要安装muduo网络库。


采用以下命令编译：




```C++
g++ server.cpp  -lmuduo_net -lmuduo_base -lpthread -o server
```
		

 


下文用poll实现非阻塞的服务器端。

			