---
layout: post
title: Linux非阻塞IO（四）非阻塞IO中connect的实现
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
我们为客户端的编写再做一些工作。
  这次我们使用非阻塞IO实现connect函数。
  

```C++
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```
		

非阻塞IO有以下用处：



  1.将三次握手的处理过程生下来，处理其他事情。


  2.使用这个同时建立多个连接。


  3.实现超时connect功能，本节实现的connect就可以指定时间，超时后算作错误处理。



 


在阻塞IO中，调用connect后一般会阻塞，直到确定连接成功或者失败。


在Non-Blocking IO中，connect往往会立刻返回，此时connect就有两种结果。



  一是连接成功




  二是返回-1，errno置为`EINPROGRESS`，这种一般是因为网络延迟，所以连接不能马上建立，我们需要使用poll来监听sockfd。



所以接下来我们需要`向poll注册sockfd的写事件`。


《TCP/IP详解》第二卷指出以下的一些规则：



  当连接建立时，sockfd可写。




  `当遇到错误时，sockfd既可读又可写`。



我们设置一个超时时间，当poll返回时，如果sockfd可写，此时有两种情况：



  一是连接确实建立成功




  二是连接发生错误



我们`需要某些手段辨别究竟是否发生了错误`。


这里我们采用socket选项，检测的是SO_ERROR，代码如下：




```C++
int get_sockfd_error(int sockfd)
{
    int err;
    socklen_t socklen = sizeof(err);
    if(getsockopt(sockfd, SOL_SOCKET, SO_ERROR, &err, &socklen) == -1)
        ERR_EXIT("getsock_error  error");
    if (err == 0)
        return 0; //无错误
    else
        return err;
}
```
		

如果sockfd无错误，则返回0，否则返回错误代码。


 


在非阻塞connect的实现中，我们通常需要先把fd设置为非阻塞，最后再重新设置为阻塞。这样做，是`为了满足阻塞与非阻塞的需求`。


因为在阻塞IO中，有时候也会使用非阻塞connect。


 


实现代码如下：




```C++
int nonblocking_connect(int sockfd, const char *des_host, uint16_t des_port, int timeout)
{
    if(des_host == NULL)
    {
        fprintf(stderr, "des_host can not be NULL\n");
        exit(EXIT_FAILURE);
    }

    SAI servaddr;
    memset(&servaddr, 0, sizeof servaddr);
    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons(des_port);
    if(inet_aton(des_host, &servaddr.sin_addr) == 0)
    {
        //DNS
        //struct hostent *gethostbyname(const char *name);
        struct hostent *hp = gethostbyname(des_host);
        if(hp == NULL)
            ERR_EXIT("gethostbyname");
        servaddr.sin_addr = *(struct in_addr*)hp->h_addr;
    }

    //设置为非阻塞
    activate_nonblock(sockfd); 

    //connect会立刻返回
    int ret = connect(sockfd, (SA*)&servaddr, sizeof servaddr);
    if(ret == -1)
    {
        if(errno != EINPROGRESS) //连接失败
            ERR_EXIT("connect error");
        struct pollfd pfd[1];
        pfd[0].fd = sockfd;
        pfd[0].events = POLLOUT;

        ret = poll(pfd, 1, timeout);

        if(ret == 0)
        {
            errno = ETIMEDOUT;
            ret = -1; //连接超时，此时判定为失败
        }
        //sockfd可写，此时需要检查套接字选项，查看是否发生错误
        else if(ret == 1 && pfd[0].revents & (POLLOUT | POLLWRBAND))
        {
            int err;
            //检查sockfd错误
            if((err = get_sockfd_error(sockfd)))
            {
                errno = err;
                return -1;
            }
        }
    }

    //重新设置为阻塞
    deactivate_nonblock(sockfd);
    return ret;
}
```
		

 


读者可以自行测试。


 


下节使用非阻塞connect实现客户端。

			