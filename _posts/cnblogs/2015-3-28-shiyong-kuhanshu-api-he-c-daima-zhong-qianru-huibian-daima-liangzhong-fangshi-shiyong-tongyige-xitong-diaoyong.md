---
layout: post
title: 使用库函数API和C代码中嵌入汇编代码两种方式使用同一个系统调用
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
本周作业的主要内容就是采用gcc嵌入汇编的方式调用system call。  
系统调用其实就是操作系统提供的服务。我们平时编写的程序，如果仅仅是数值计算，那么所有的过程都是在用户态完成的，但是我们想将变量打印在屏幕上，就必须调用printf，而printf这个函数内部就使用了write这个系统调用。  
操作系统之所以以system call的方式提供服务，是因为如果程序员能够任意操作OS所有的资源，那么将无比危险，所以OS设计出了内核态和用户态。
我们平时编程都是在用户态下，如果我们想要调用系统资源，那么就必须采用系统调用，陷入内核态，才能达到目的。


下面我们采用几个例子，按照由浅入深的方式一一说明。

##getpid 简单示例

getpid的函数很简单，就是获取当前进程的进程号。
使用C调用如下：

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
int main(int argc, const char *argv[])
{
    pid_t tt;
    tt = getpid();
    printf("%u\n", tt);
    return 0;
}

```

使用内嵌汇编调用如下：

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
int main(int argc, const char *argv[])
{
    pid_t tt;
    asm volatile(
            "mov $0x14, %%eax\n\t"
            "int $0x80\n\t"
            "mov %%eax, %0\n\t"
            :"=m" (tt)
    		);
    printf("%u\n", tt);
    return 0;
}
```

内嵌汇编在语法上要求先声明输出参数，然后声明输出参数值。上述代码中getpid不需要参数，只需要一个输出值。
对于内嵌汇编调用system call：

	1.系统调用号放在eax中。
	2.系统调用的参数，按照顺序分别放在ebx、ecx、edx、esi、edi和ebp中
	3.返回值使用eax传递

上面的代码之所以使用中断，是因为中断（包括异常）是进入到内核态的唯一方式。
所以我们使用int 0x80触发中断，然后中断处理程序保存现场，我们的进程陷入内核态。

其实，使用系统调用还有一种方式：

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/syscall.h>

int main(int argc, const char *argv[])
{
    pid_t tt;
    tt = syscall(SYS_getpid);
    printf("%u\n", tt);
    return 0;
}
```

这种方式其实内部也是采用的内嵌汇编。  
linux中有个函数叫做gettid，这个函数用来取出当前线程的pid（Linux中的线程是使用进程模拟实现的，所以每个线程都有一个全局唯一的pid），可以查到它的声明，但是使用时，编译报错，提示函数找不到，因为libc中没有提供这个函数。此时我们就可以借助这种方式，使用syscall(SYS_gettid)即可。

##fork 使用

fork函数同样不需要参数，只有输出，这里给出两个版本：

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>


int main(int argc, const char *argv[])
{
    pid_t tt;
    tt = fork();
    if (tt == 0)
    {
        printf("子进程\n");

    }
    else
    {
        printf("父进程\n");
    }
    printf("%u\n", tt);
    return 0;
}
```
fork这个函数有个特点，就是调用一次返回两次，原因在于它复制出了一个子进程，执行同样地代码段。  
区分子进程和父进程的手段就是检查返回值。

下面是使用汇编的版本

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>


pid_t _fork()
{
    pid_t tt;
    asm volatile(
            "mov $0x2, %%eax\n\t"
            "int $0x80\n\t"
            "mov %%eax, %0\n\t"
            :"=m" (tt)
            );
    return tt;
}


int main(int argc, const char *argv[])
{
    pid_t tt;
    tt = _fork();
    if (tt == 0)
    {
        printf("child\n");

    }
    else
    {
        printf("parent\n");
    }
    printf("%u\n", tt);
    return 0;
}
```


##read

上面的getpid和fork都不需要参数，下面我们看下read函数如何使用汇编调用。
read的函数声明如下：

```C
ssize_t read(int fd, void *buf, size_t count);
```

read函数需要三个参数。
下面我们看下它的过程，这里为了减少篇幅，不再贴出完整的main函数。

```C
ssize_t _read(int fd, void *buf, size_t count)
{
    int ret;
    asm volatile(
            "mov %3, %%edx\n\t"  // count->edx
            "mov %2, %%ecx\n\t"  // buf->ecx
            "mov %1, %%ebx\n\t"  // fd->ebx
            "mov $0x3, %%eax\n\t"
            "int $0x80\n\t"
            "mov %%eax, %0\n\t"
            :"=m"(ret)
            :"b"(fd), "c"(buf), "d"(count)
            );
    return ret;

}
```
上面提过，参数保存在ebx、ecx等寄存器中，这里的三个参数就是放在这三个寄存器中。最后一行的

	:"b"(fd), "c"(buf), "d"(count)
	
就是声明，fd使用的是ebx，buf使用ecx传递，count使用edx传递。


##使用timerfd的定时器

下面是一个较复杂的示例，它使用了timerfd系列的定时器，timefd是Linux2.6之后加入的新的系统调用。它将定时器的触发采用fd可读这个事件来表现。所以对于timerfd，我们可以使用epoll，将它和socket的fd一同监视。

C源码如下：

```C
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/timerfd.h>
#define ERR_EXIT(m) \
    do { \
        perror(m);\
        exit(EXIT_FAILURE);\
    }while(0)

int main(int argc, const char *argv[])
{
    //创建定时器的fd
    int timerfd = timerfd_create(CLOCK_REALTIME, 0);
    if(timerfd == -1)
        ERR_EXIT("timerfd_create");

    //开启定时器，并设置时间
    struct itimerspec howlong;
    memset(&howlong, 0, sizeof howlong);
    howlong.it_value.tv_sec = 4;  //初始时间
    howlong.it_interval.tv_sec = 1; //间隔时间
    if(timerfd_settime(timerfd, 0, &howlong, NULL) == -1)
        ERR_EXIT("timerfd_settime");

    int ret;
    char buf[1024];

    while((ret = read(timerfd, buf, sizeof buf)) > 0)
    {
        printf("foobar ....\n");
    }

    close(timerfd);


    return 0;
}
```

这段代码主要是注册了一个定时器的fd，然后设置初始化时间和事件触发间隔，然后每隔几秒钟，该定时器fd变为可读。  
这段代码的运行效果是：先运行4s，然后每隔1s打印出一行"foobar ..."

下面是使用汇编的版本：

```C
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/timerfd.h>
#define ERR_EXIT(m) \
    do { \
        perror(m);\
        exit(EXIT_FAILURE);\
    }while(0)

int _timerfd_create(int clockid, int flags);
int _timerfd_settime(int fd, int flags,
        const struct itimerspec *new_value,
        struct itimerspec *old_value);
ssize_t _read(int fd, void *buf, size_t count);
ssize_t _write(int fd, const void *buf, size_t count);
int _close(int fd);


int main(int argc, const char *argv[])
{
    //创建定时器的fd
    int timerfd = _timerfd_create(CLOCK_REALTIME, 0);
    if(timerfd == -1)
        ERR_EXIT("timerfd_create");

    //开启定时器，并设置时间
    struct itimerspec howlong;
    memset(&howlong, 0, sizeof howlong);
    howlong.it_value.tv_sec = 2;  //初始时间
    howlong.it_interval.tv_sec = 1; //间隔时间
    if(_timerfd_settime(timerfd, 0, &howlong, NULL) == -1)
        ERR_EXIT("timerfd_settime");

    int ret;
    char buf[1024];

    while((ret = _read(timerfd, buf, sizeof buf)) > 0)
    {
        printf("foobar ....\n");
    }

    close(timerfd);


    return 0;
}


ssize_t _read(int fd, void *buf, size_t count)
{
    int ret;
    asm volatile(
            "mov %3, %%edx\n\t"  // len->edx
            "mov %2, %%ecx\n\t"  // str->ecx
            "mov %1, %%ebx\n\t"  // fd->ebx
            "mov $0x3, %%eax\n\t"
            "int $0x80\n\t"
            "mov %%eax, %0\n\t"
            :"=m"(ret)
            :"b"(fd), "c"(buf), "d"(count)
            );
    return ret;

}


ssize_t _write(int fd, const void *buf, size_t count)
{
    int ret;
    asm volatile(
            "mov %3, %%edx\n\t"  // len->edx
            "mov %2, %%ecx\n\t"  // str->ecx
            "mov %1, %%ebx\n\t"  // fd->ebx
            "mov $0x4, %%eax\n\t"
            "int $0x80\n\t"
            "mov %%eax, %0\n\t"
            :"=m"(ret)
            :"b"(fd), "c"(buf), "d"(count)
            );
    return ret;
}


int _close(int fd)
{
    int ret;
    asm volatile(
            "mov %1, %%ebx\n\t"  // fd->ebx
            "mov $0x6, %%eax\n\t"
            "int $0x80\n\t"
            "mov %%eax, %0\n\t"
            :"=m"(ret)
            :"b"(fd)
            );
    return ret;

}


int _timerfd_create(int clockid, int flags)
{
    int ret;
    asm volatile(
            "mov %2, %%ecx\n\t" // flags
            "mov %1, %%ebx\n\t"  // clockid
            "mov $322, %%eax\n\t"
            "int $0x80\n\t"
            "mov %%eax, %0\n\t"
            :"=m"(ret)
            :"b"(clockid), "c"(flags)
            );
    return ret;

}


int _timerfd_settime(int fd, int flags,
        const struct itimerspec *new_value,
        struct itimerspec *old_value)
{
    int ret;
    asm volatile(
            "mov %4, %%esi\n\t"  // old value
            "mov %3, %%edx\n\t"  // len->edx
            "mov %2, %%ecx\n\t"  // str->ecx
            "mov %1, %%ebx\n\t"  // fd->ebx
            "mov $325, %%eax\n\t"
            "int $0x80\n\t"
            "mov %%eax, %0\n\t"
            :"=m"(ret)
            :"b"(fd), "c"(flags), "d"(new_value), "S"(old_value)
            );
    return ret;

}
```

这里注意下

timerfd_settime这个系统调用共四个参数，最后一个参数采用的esi，采用“S”说明。  
查看某个系统调用号，例如timerfd系列，可以使用如下的命令：

	➜  ~  cat /usr/include/i386-linux-gnu/asm/unistd_32.h | grep timerfd
	#define __NR_timerfd_create 322
	#define __NR_timerfd_settime 325
	#define __NR_timerfd_gettime 326


##实验截图
![](http://images.cnitblog.com/blog2015/669654/201503/281627439429907.png)
![](http://images.cnitblog.com/blog2015/669654/201503/281628105674742.png)




##系统调用原理

通过本周的作业，更加熟悉了系统调用的本质，以及系统调用和中断的关联。系统调用是用户态和内核态的桥梁，而具体的措施就是中断。 
上面我们采用内嵌汇编编写的代码，在运行时，通过eax准备系统调用号，使用ebx、ecx等传递具体参数，当我们触发0x80中断时，经过中断处理程序，我们就进入了内核态。 

##作业署名

郭春阳 原创作品转载请注明出处 ：[《Linux内核分析》MOOC课程](http://mooc.study.163.com/course/USTC-1000029000#/info)
			