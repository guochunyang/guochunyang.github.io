---
layout: post
title: Linux非阻塞IO（三）非阻塞IO中缓冲区Buffer的实现
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
本文我们来实现回射服务器的Buffer。
   
  
###Buffer的实现
   
  上节`提到了非阻塞IO必须具备Buffer`。再次将Buffer的设计描述一下：
  <a href="http://images.cnitblog.com/blog/669654/201410/241535338248737.png"><img style="background-image: none; border-bottom: 0px; border-left: 0px; padding-left: 0px; padding-right: 0px; display: inline; border-top: 0px; border-right: 0px; padding-top: 0px" title="buffer1" border="0" alt="buffer1" src="http://images.cnitblog.com/blog/669654/201410/241535401216147.png" width="923" height="349" /></a>
  这里必须补充一点，`writeIndex指向空闲空间的第一个位置。`
  `这里有<font color="#ff0000">三个重要的不变式`：</font>
     1. 0 <= readIndex <= writeIndex <= BUFFER_SIZE
      2. writeIndex – readIndex 为可以从buffer读取的字节数
      3. BUFFER_SIZE – writeIndex 为buffer还可以继续读取的字节数
      还有一点，数据读取完毕之后，要重置下标为0
   根据我设计的这个示意图，我利用结构体封装了一个Buffer，如下：
  

```Python
#ifndef BUFFER_H_
#define BUFFER_H_

#include <poll.h>

#define BUFFER_SIZE 1024

typedef struct {
    char buf_[BUFFER_SIZE];
    int readIndex_; //读取数据
    int writeIndex_; //写入数据
} buffer_t;

void buffer_init(buffer_t *bt);
int buffer_is_readable(buffer_t *bt);
int buffer_is_writeable(buffer_t *bt);
int buffer_read(buffer_t *bt, int sockfd);
int buffer_write(buffer_t *bt, int sockfd);

#define kReadEvent (POLLIN | POLLPRI)
#define kWriteEvent (POLLOUT | POLLWRBAND)

#endif //BUFFER_H_
```
		

这里的buffer先采用固定长度，后期可以改为动态数组。


下面我们来实现Buffer的每个函数。


第一个是初始化，内存清零，下标都设置为0即可。




```C++
void buffer_init(buffer_t *bt)
{
    memset(bt->buf_, 0, sizeof(bt->buf_));
    bt->readIndex_ = 0;
    bt->writeIndex_ = 0;
}
```
		

缓冲区是否可以读出数据，需要`判断(writeIndex – readIndex)是否大于0`




```C++
int buffer_is_readable(buffer_t *bt)
{
    return bt->writeIndex_ > bt->readIndex_;
}
```
		



缓冲区是否可写，需要判断是否有空闲空间。




```C++
int buffer_is_writeable(buffer_t *bt)
{
    return BUFFER_SIZE > bt->writeIndex_;
}
```
		

接下来是调用read函数，buffer从fd中读取数据，read的最后一个参数为buffer的剩余空间。




```C++
int buffer_read(buffer_t *bt, int sockfd)
{
    int nread = read(sockfd, &bt->buf_[bt->writeIndex_], BUFFER_SIZE - bt->writeIndex_);
    if(nread == -1)
    {
        if(errno != EWOULDBLOCK)
            ERR_EXIT("read fd error");
        return -1;
    }
    else
    {
        bt->writeIndex_ += nread;
        return nread;
    }
}
```
		



最后是输出操作，将buffer中的数据写入sockfd，write的最后一个参数为buffer现存的字节数。




```C++
int buffer_write(buffer_t *bt, int sockfd)
{
    int nwriten = write(sockfd, &bt->buf_[bt->readIndex_], bt->writeIndex_ - bt->readIndex_);
    if(nwriten == -1)
    {
        if(errno != EWOULDBLOCK)
            ERR_EXIT("write fd error");
        return -1;
    }
    else
    {
        bt->readIndex_ += nwriten;
        if(bt->readIndex_ == bt->writeIndex_)
        {
            bt->readIndex_ = bt->writeIndex_ = 0;
        }
        return nwriten;
    }
}
```
		

 


Buffer的实现完毕。


 


下文开始编写回射服务器的客户端。

			