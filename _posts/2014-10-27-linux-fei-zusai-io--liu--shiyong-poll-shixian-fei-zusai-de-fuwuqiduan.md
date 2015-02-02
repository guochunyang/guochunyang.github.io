---
layout: post
title: Linux非阻塞IO（六）使用poll实现非阻塞的服务器端
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
关于poll模型监听的事件以及返回事件，我们定义宏如下：
  

```Python
#define kReadEvent (POLLIN | POLLPRI)
#define kWriteEvent (POLLOUT | POLLWRBAND)
#define kReadREvent (POLLIN | POLLPRI | POLLRDHUP)
#define kWriteREvent (POLLOUT)
```
		

前面我们说明了，为什么非阻塞IO必须具备缓冲区。事实上，对于server而言，每条TCP连接应该具有两个缓冲区，一个用于输入，一个用于输出。



  sockfd –>  `InputBuffer` –> 用户空间 –> 处理数据 –> 得到结果 –> `OutputBuffer` –> sockfd



但是`本例仅仅是简单的回射服务`，收到数据立刻发出，所以我们只使用一个缓冲区。


对于每个TCP连接，我们对应这样的一个数据结构：




```Python
typedef struct{
    buffer_t buffer_;
} tcp_connection_t; //表示一条TCP连接
```
		

那么我们需要建立一个从fd到对应的tcp_connection_t的对应关系。我使用以下的数组：




```C++
tcp_connection_t *connsets[EVENTS_SIZE]; //提供从fd到TCP连接的映射
```
		

这个数组初始化为NULL，然后我使用fd作为下标，也就是说，当我需要处理某fd的时候，`以该fd为下标`就可以找到它相应的tcp_connection_t指针。`这本质上是一种哈希表的思想`。


使用tcp_connection_t的逻辑是：



  每当accept一个新的连接，就动态创建一个新的tcp_connection_t，并且`将其指针保存至以peerfd为下标的位置中`。




  每当连接关闭，需要释放内存，同时重置指针为NULL。



我将所有的代码全部写入main函数中，因为前面封装了buffer，代码的可读性已经提高，再加上我们的业务逻辑简单，进一步封装，反而降低可读性。


这里跟client有几处相同点：



  仅当缓冲区有空闲空间时才监听read事件




  仅当缓冲区有数据时，才监听write事件



所以我们每次进行poll调用前都需要将fd的监听事件重新装填。


server的代码整体框架如下：




```C++
//初始化poll

while(1)
{
    //重新装填fd数组

    //poll系统调用

    //依次处理每个fd的读写事件
}
```
		

所以完整的代码如下：




```Python
#define _GNU_SOURCE             /* See feature_test_macros(7) */
#include <sys/socket.h>
#include "sysutil.h"
#include "buffer.h"
#include <assert.h>

#define EVENTS_SIZE 1024

typedef struct{
    buffer_t buffer_;
} tcp_connection_t; //表示一条TCP连接

tcp_connection_t *connsets[EVENTS_SIZE]; //提供从fd到TCP连接的映射

int main(int argc, char const *argv[])
{
    //获取监听fd
    int listenfd = tcp_server("localhost", 9981);
    //将监听fd设置为非阻塞
    activate_nonblock(listenfd);

    //初始化connsets
    int i;
    for(i = 0; i < EVENTS_SIZE; ++i)
    {
        connsets[i] = NULL;
    }

    //初始化poll
    struct pollfd events[EVENTS_SIZE];
    for(i = 0; i < EVENTS_SIZE; ++i)
        events[i].fd = -1;
    events[0].fd = listenfd;
    events[0].events = kReadEvent;
    int maxi = 0;

    while(1)
    {
        //重新装填events数组
        int i;
        for(i = 1; i < EVENTS_SIZE; ++i)
        {
            int fd = events[i].fd;
            events[i].events = 0; //重置events
            if(fd == -1)
                continue;
            assert(connsets[fd] != NULL);
            
            //当Buffer中有数据可读时，才监听write事件
            if(buffer_is_readable(&connsets[fd]->buffer_))
            {
                events[i].events |= kWriteEvent;
            }

            //当Buffer中有空闲空间时，才监听read事件
            if(buffer_is_writeable(&connsets[fd]->buffer_))
            {
                events[i].events |= kReadEvent;
            }

        }

        //poll调用
        int nready = poll(events, maxi + 1, 5000);
        if(nready == -1)
            ERR_EXIT("poll");
        else if(nready == 0)
        {
            printf("poll timeout.\n");
            continue;
        }

        
        //处理listenfd
        if(events[0].revents & kReadEvent)
        {
            //接受一个新的客户fd
            int peerfd = accept4(listenfd, NULL, NULL, SOCK_NONBLOCK | SOCK_CLOEXEC);
            if(peerfd == -1)
                ERR_EXIT("accept4");
            //新建tcp连接
            tcp_connection_t *pt = (tcp_connection_t*)malloc(sizeof(tcp_connection_t));
            buffer_init(&pt->buffer_);
            //将该tcp连接放入connsets
            connsets[peerfd] = pt;
            //放入events数组
            int i;
            for(i = 0; i < EVENTS_SIZE; ++i)
            {
                if(events[i].fd == -1)
                {
                    events[i].fd = peerfd; //这里不必监听fd
                    if(i > maxi)
                        maxi = i; //更新maxi
                    break;
                }
            }
            if(i == EVENTS_SIZE)
            {
                fprintf(stderr, "too many clients\n");
                exit(EXIT_FAILURE);
            }
        }

        //处理客户fd
        //int i;
        for(i = 1; i <= maxi; ++i)
        {
            int sockfd = events[i].fd;
            if(sockfd == -1)
                continue;
            //取出指针
            tcp_connection_t *pt = connsets[sockfd];
            assert(pt != NULL);
            if(events[i].revents & kReadREvent) //读取数据
            {
                if(buffer_read(&pt->buffer_, sockfd) == 0)
                {
                    //close
                    events[i].fd = -1;
                    close(sockfd);
                    free(pt);
                    connsets[sockfd] = NULL;
                    continue; //继续下一次循环
                } 
            }

            if(events[i].revents & kWriteREvent) //可以发送数据
            {
                buffer_write(&pt->buffer_, sockfd);
            }
        }
    }

    close(listenfd);

    return 0;
}
```
		

需要注意的是，关于accept，我使用的是`accpet4，这是Linux新增的系统调用`：


 




```C++
int accept4(int sockfd, struct sockaddr *addr, socklen_t *addrlen, int flags);
```
		

与accept的区别就是accept4可以指定非阻塞标志位。


 


下文使用epoll。

			