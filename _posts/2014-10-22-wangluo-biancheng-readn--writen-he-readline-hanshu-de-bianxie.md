---
layout: post
title: 网络编程readn、writen和readline函数的编写
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---

###readn
   
  在Linux中，read的声明为：
  

```C++
ssize_t read(int fd, void *buf, size_t count);
```
		

它的返回值有以下情形：



  1.大于0，代表成功读取的字节数


  2.等于0，代表读取到了EOF，一般是对方关闭了socket的写端或者直接close


  3.小于0，出现错误。





我们编写一个readn函数，声明与read一致，但是，`readn在未出错或者fd没有关闭的情况下，会读满count个字节`。




```C++
ssize_t readn(int fd, void *buf, size_t count)
{
    size_t nleft = count;  //剩余的字节数
    ssize_t nread; //用作返回值
    char *bufp = (char*)buf; //缓冲区的偏移量

    while(nleft > 0)
    {
        nread = read(fd, bufp, nleft);
        if(nread == -1)
        {
            if(errno == EINTR)
                continue;
            return -1; // ERROR
        }
        else if(nread == 0) //EOF
            break;

        nleft -= nread;
        bufp += nread;
    }

    return (count - nleft);
}
```
		

readn的返回值含义如下：



  1.小于0，出错


  2.等于0，对方关闭


  3.大于0，但是小于count，对方关闭


  4.count，代表读满count个字节



 



###writen


 


write函数的声明如下：




```C++
ssize_t write(int fd, const void *buf, size_t count);
```
		



man手册中对write的返回值描述如下：


       On success, the number of bytes written is returned (zero indicates nothing was  writ‐
  <br />       ten).  On error, -1 is returned, and errno is set appropriately.


       If  count  is  zero and fd refers to a regular file, then write() may return a failure
  <br />       status if one of the errors below is detected.  If no errors are detected, 0  will  be

  <br />       returned  without  causing any other effect.  If count is zero and fd refers to a file

  <br />       other than a regular file, the results are not specified.


解释如下：



  成功时，返回成功写入的字节数，否则返回-1，并设置相应的errno。


  如果count为0，并且fd指向一个普通文件，那么当探测到错误时返回-1.如果没有错误发生，返回0，不会产生任何影响。


  如果count为0，并且fd指向的不是普通文件，那么结果未定义。



我们不去追究write为0的情形。编写write如下：




```C++
ssize_t writen(int fd, const void *buf, size_t count)
{
    size_t nleft = count;
    ssize_t nwrite;
    const char *bufp = (const char*)buf;
    
    while(nleft > 0)
    {
        nwrite = write(fd, bufp, nleft);
        if(nwrite <= 0) // ERROR
        {
            if(nwrite == -1 && errno == EINTR)
                continue;
            return -1;
        }

        nleft -= nwrite;
        bufp += nwrite;
    }
    
    return count;
}
```
		

从代码中可以看出，`writen要么写满count字节，要么失败`。


 



###readline


 


在网络编程中，很多协议是基于文本行的，例如HTTP和FTP，还有telnet，他们的消息每行都是以\r\n作为结束标志的。于是我们开发一个readline函数，声明如下：




```C++
ssize_t readline(int sockfd, void *usrbuf, size_t maxlen)
```
		

readline函数的语义是：



  如果碰不到\n，那么读取maxlen-1个字节，最后一个位置补充\0。


  否则读取到\n，在后面加一个\0。如果中间遇到EOF，`直接返回0，而不是已经读取的字节数`。



我们先给出一种低效的实现：




```C++
ssize_t readline_slow(int fd, void *usrbuf, size_t maxlen)
{
    char *bufp = usrbuf;  //记录缓冲区当前位置
    ssize_t nread;
    size_t nleft = maxlen - 1;  //留一个位置给 '\0'
    char c;
    while(nleft > 0)
    {
        if((nread = read(fd, &c, 1)) < 0)
        {
            if(errno == EINTR)
                continue;
            return -1;
        }else if(nread == 0) // EOF
        {
            break;
        }

        //普通字符
        *bufp++ = c;
        nleft--;

        if(c == '\n')
            break;
    }
    *bufp = '\0';
    return (maxlen - nleft - 1);
}
```
		



这个的思路很简单，每次读取一个字节，直到遇到换行符为止。


这种实现是低效的，因为每次读取一个字节，都要进行一次系统调用。


在网络编程中，还有一个函数叫做recv，如下：




```C++
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
```
		



它相对于read，多了一个flags选项。


有一个选项为MSG_PEEK，描述如下：


This flag causes the receive operation to return data from the beginning of the
  <br />receive queue without removing that data from the queue.   Thus,  a  subsequent

  <br />receive call will return the same data.


大致意思是它从内核中读取数据，但并不会将数据移除，所以`这个flag起到了一个预览内核数据的作用`。这样我们就可以先从内核中读取一大块数据，检查其中是否存在\n，如果不存在，这么将这些数据全部读取，如果存在，则读取到\n为止。


我们先实现recv_peek函数：




```C++
ssize_t recv_peek(int sockfd, void *buf, size_t len)
{
    int nread;
    do
    {
        nread = recv(sockfd, buf, len, MSG_PEEK);
    }
    while(nread == -1 && errno == EINTR);

    return nread;
}
```
		



readline函数的实现如下：




```C++
ssize_t readline(int sockfd, void *usrbuf, size_t maxlen)
{
    //
    size_t nleft = maxlen - 1;
    char *bufp = usrbuf; //缓冲区位置
    size_t total = 0; //读取的字节数

    ssize_t nread;
    while(nleft > 0)
    {
        //预读取
        nread = recv_peek(sockfd, bufp, nleft);
        if(nread <= 0)
            return nread;

        //检查\n
        int i;
        for(i = 0; i < nread; ++i)
        {
            if(bufp[i] == '\n')
            {
                //找到\n
                size_t nsize = i+1;
                if(readn(sockfd, bufp, nsize) != nsize)
                    return -1;
                bufp += nsize;
                total += nsize;
                *bufp = 0;
                return total;
            }
        }

        //没找到\n
        if(readn(sockfd, bufp, nread) != nread)
            return -1;
        bufp += nread;
        total += nread;
        nleft -= nread;
    }
    *bufp = 0;
    return maxlen - 1;
}
```
		

 


我们编写的这三个函数后面可以用于处理TCP分包问题，后面写文章叙述。

			