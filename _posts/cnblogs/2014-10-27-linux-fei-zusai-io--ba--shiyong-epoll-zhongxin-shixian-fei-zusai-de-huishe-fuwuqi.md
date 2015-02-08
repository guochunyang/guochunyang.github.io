---
layout: post
title: Linux非阻塞IO（八）使用epoll重新实现非阻塞的回射服务器
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
本文无太多内容，主要是几个前面提到过的注意点：
  一是epoll的fd需要重新装填。我们将tcp_connection_t的指针保存在数组中，所以我们`以这个数组为依据，重新装填fd的监听事件`。
  

```C++
//重新装填epoll内fd的监听事件
        int i;
        for(i = 0; i < EVENTS_SIZE; ++i)
        {
            if(connsets[i] != NULL)
            {
                int fd = i; //fd
                tcp_connection_t *pt = connsets[i]; //tcp conn
                uint32_t event = 0;
                if(buffer_is_readable(&pt->buffer_))
                    event |= kWriteEvent;
                if(buffer_is_writeable(&pt->buffer_))
                    event |= kReadEvent;
                //重置监听事件
                epoll_mod_fd(epollfd, fd, event);
            }
        }
```
		



二是，建立连接时，需要做的工作是：



  1.新建tcp_connection_t结构，初始化


  2.将fd加入epoll，`不监听任何事件`


  3.将tcp_connection_t的指针加入数组。



代码如下：


<div style="border-bottom: #cccccc 1px solid; border-left: #cccccc 1px solid; padding-bottom: 5px; background-color: #f5f5f5; padding-left: 5px; padding-right: 5px; border-top: #cccccc 1px solid; border-right: #cccccc 1px solid; padding-top: 5px" class="cnblogs_code">
  

```C++
//建立连接
int peerfd = accept4(listenfd, NULL, NULL, SOCK_NONBLOCK | SOCK_CLOEXEC);
if(peerfd == -1)
      ERR_EXIT("accept4");
//新建tcp连接
tcp_connection_t *pt = (tcp_connection_t*)malloc(sizeof(tcp_connection_t));
buffer_init(&pt->buffer_);
//将该tcp连接放入connsets
connsets[peerfd] = pt;
epoll_add_fd(epollfd, peerfd, 0);
```
		
</div>

连接关闭时需要：




```C++
//close
epoll_del_fd(epollfd, fd);
close(fd);
free(pt);
connsets[fd] = NULL;
```
		



 


还有一点：前面我们记录fd和connsets的关系，采用的是数组下标的方式，其实我们`还可以将指针存入epoll的data`中，其中：




```Python
typedef union epoll_data {
   void        *ptr;
   int          fd;
   uint32_t     u32;
   uint64_t     u64;
} epoll_data_t;

struct epoll_event {
   uint32_t     events;      /* Epoll events */
   epoll_data_t data;        /* User data variable */
};
```
		

我们对于data这个联合体，`不再使用fd，而是使用ptr，指向一个tcp_connection_t的指针`。不过我们需要将fd存储在tcp_connection_t数据结构中。


这里为了简便起见，仍采用以前的方法，读者可以自行尝试。


 


完整的代码如下：




```Python
#define _GNU_SOURCE             /* See feature_test_macros(7) */
#include <sys/socket.h>
#include "sysutil.h"
#include "buffer.h"
#include <assert.h>
#include <sys/epoll.h>

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
    int ix;
    for(ix = 0; ix < EVENTS_SIZE; ++ix)
    {
        connsets[ix] = NULL;
    }


    //初始化epoll
    int epollfd = epoll_create1(0);
    epoll_add_fd(epollfd, listenfd, kReadEvent);
    struct epoll_event events[1024];


    while(1)
    {
        //重新装填epoll内fd的监听事件
        int i;
        for(i = 0; i < EVENTS_SIZE; ++i)
        {
            if(connsets[i] != NULL)
            {
                int fd = i; //fd
                tcp_connection_t *pt = connsets[i]; //tcp conn
                uint32_t event = 0;
                if(buffer_is_readable(&pt->buffer_))
                    event |= kWriteEvent;
                if(buffer_is_writeable(&pt->buffer_))
                    event |= kReadEvent;
                //重置监听事件
                epoll_mod_fd(epollfd, fd, event);
            }
        }

        //epoll监听fd
        int nready = epoll_wait(epollfd, events, 1024, 5000);
        if(nready == -1)
            ERR_EXIT("epoll wait");
        else if(nready == 0)
        {
            printf("epoll timeout.\n");
            continue;
        }

        //处理fd
        for(i = 0; i < nready; ++i)
        {
            int fd = events[i].data.fd;
            uint32_t revents = events[i].events;
            if(fd == listenfd) //处理listen fd
            {
                if(revents & kReadREvent)
                {
                    //建立连接
                    int peerfd = accept4(listenfd, NULL, NULL, SOCK_NONBLOCK | SOCK_CLOEXEC);
                    if(peerfd == -1)
                        ERR_EXIT("accept4");
                    //新建tcp连接
                    tcp_connection_t *pt = (tcp_connection_t*)malloc(sizeof(tcp_connection_t));
                    buffer_init(&pt->buffer_);
                    //将该tcp连接放入connsets
                    connsets[peerfd] = pt;
                    epoll_add_fd(epollfd, peerfd, 0);
                }
            }
            else //处理普通客户的fd
            {
                //取出指针
                tcp_connection_t *pt = connsets[fd];
                assert(pt != NULL);
                if(revents & kReadREvent)
                {
                    if(buffer_read(&pt->buffer_, fd) == 0)
                    {
                        //close
                        epoll_del_fd(epollfd, fd);
                        close(fd);
                        free(pt);
                        connsets[fd] = NULL;
                        continue; //继续下一次循环
                    } 
                }

                if(revents & kWriteREvent)
                {
                    buffer_write(&pt->buffer_, fd);
                }
            }
        }
    }

    close(listenfd);

    return 0;
}
```
		

 


下文使用epoll的ET模式。

			