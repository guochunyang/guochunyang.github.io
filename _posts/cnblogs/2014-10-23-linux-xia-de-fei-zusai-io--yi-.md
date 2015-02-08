---
layout: post
title: Linux下的非阻塞IO（一）
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
非阻塞IO是相对于传统的阻塞IO而言的。
  我们首先需要搞清楚，什么是阻塞IO。APUE指出，系统调用分为两类，低速系统调用和其他，`其中低速系统调用是可能会使进程永远阻塞的一类系统调用`。但是与磁盘IO有关的系统调用是个例外。
  我们以read和write为例，read函数读取stdin，如果是阻塞IO，那么：
     如果我们不输入数据，那么read函数会一直阻塞，一直到我们输入数据为止。
   如果是非阻塞IO，那么：
     如果存在数据，读取然后返回，如果没有输入，那么直接返回-1，errno置为EAGAIN
   我们用write做一个实验：
  

```C++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <sys/wait.h>
#include <errno.h>
#include <signal.h>

char buf[500000];

int main(int argc, const char *argv[])
{
    int ntowrite, nwrite;

    ntowrite = read(STDIN_FILENO, buf, sizeof buf);
    fprintf(stderr, "read %d bytes\n", ntowrite);

    activate_nonblock(STDOUT_FILENO, O_NONBLOCK);

    char *ptr = buf;
    int nleft = ntowrite; //剩余的字节数
    while(nleft > 0)
    {
        errno = 0;
        nwrite = write(STDOUT_FILENO, ptr, nleft);
        fprintf(stderr, "nwrite = %d, errno = %d\n", nwrite, errno);

        if(nwrite > 0)
        {
            ptr += nwrite;
            nleft -= nwrite;
        }
    }

    deactivate_nonblock(STDOUT_FILENO);

    return 0;
}
```
		

该程序向标准输出写入500000个字节。


如果使用：
  

```C++
./test < test.mkv > temp.file
```
		



那么输出结果为：




```C++
read 500000 bytes
nwrite = 500000, errno = 0
```
		

因为磁盘IO的速度较快，所以一次就可以写入，下面我们使用终端：




```C++
./test < test.mkv  2> stderr.txt
```
		

这行命令将500000的内容打印到屏幕上，同时将fprintf记录的信息通过标准错误流写入stderr.txt。


我们查看stderr.txt文件：




```C++
read 500000 bytes
nwrite = 12708, errno = 0
nwrite = -1, errno = 11
nwrite = -1, errno = 11
nwrite = -1, errno = 11
nwrite = -1, errno = 11
nwrite = -1, errno = 11
nwrite = -1, errno = 11
nwrite = 11687, errno = 0
nwrite = -1, errno = 11</pre>

  <pre>…………………………………………..</pre>

  nwrite = -1, errno = 11
    <br />nwrite = -1, errno = 11

    <br />nwrite = -1, errno = 11

    <br />nwrite = -1, errno = 11

    <br />nwrite = -1, errno = 11

    <br />nwrite = -1, errno = 11

    <br />nwrite = -1, errno = 11

    <br />nwrite = -1, errno = 11

    <br />nwrite = -1, errno = 11

    <br />nwrite = 1786, errno = 0

</div>

采用命令统计了一下，总计read次数为15247次，其中返回-1次数为15203次，说明成功读取次数为44次。


上面例子中，这种采用非阻塞IO的方式称为“轮询”，显然这是一种低效的方式，非阻塞IO通常与IO复用模型结合使用。


 


另外，将fd设置为阻塞和非阻塞的函数代码如下：


<div style="border-bottom: #cccccc 1px solid; border-left: #cccccc 1px solid; padding-bottom: 5px; background-color: #f5f5f5; padding-left: 5px; padding-right: 5px; border-top: #cccccc 1px solid; border-right: #cccccc 1px solid; padding-top: 5px" class="cnblogs_code">
  <pre>void activate_nonblock(int fd)
{
    int ret;
    int flags = fcntl(fd, F_GETFL);
    if (flags == -1)
        ERR_EXIT("fcntl");
    flags |= O_NONBLOCK;
    ret = fcntl(fd, F_SETFL, flags);
    if (ret == -1)
        ERR_EXIT("fcntl");
}


void deactivate_nonblock(int fd)
{
    int ret;
    int flags = fcntl(fd, F_GETFL);
    if (flags == -1)
        ERR_EXIT("fcntl");
    flags &= ~O_NONBLOCK;
    ret = fcntl(fd, F_SETFL, flags);
    if (ret == -1)
        ERR_EXIT("fcntl");
}
```
		

 


未完待续。

			