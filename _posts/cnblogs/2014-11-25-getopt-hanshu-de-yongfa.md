---
layout: post
title: getopt函数的用法
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
Linux提供了一个解析命令行参数的函数。
  

```C++
#include <unistd.h>

       int getopt(int argc, char * const argv[],
                  const char *optstring);

       extern char *optarg;
       extern int optind, opterr, optopt;
```
		

使用这个函数，我们可以这样运行命令




```C++
./a.out -n -t 100
```
		

n后面不需要参数，t需要一个数值作为参数。


代码如下：




```Python
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#define ERR_EXIT(m) \
    do { \
        perror(m);\
        exit(EXIT_FAILURE);\
    }while(0)


int main(int argc, char *argv[])
{
    int opt;
    while(1)
    {
        opt = getopt(argc, argv, "nt:");
        if(opt == '?')
            exit(EXIT_FAILURE);
        if(opt == -1)
            break;

        switch(opt)
        {
            case 'n':
                printf("AAAAAAAA\n");
                break;
            case 't':
                printf("BBBBBBBB\n");
                int n = atoi(optarg);
                printf("n = %d\n", n);
        }
    }
    return 0;
}
```
		

当输入非法参数时，getopt返回’?’，当解析完毕时，返回-1.


如果需要参数，那么使用optarg获取，这是一个全局变量。


注意getopt的第三个参数”nt:”，说明可用的参数有n和t，t后面有一个冒号，说明t需要额外的参数。


运行结果如下：




```C++
5:30:22 wing@ubuntu msg ./getopt_test -n
AAAAAAAA
5:30:26 wing@ubuntu msg ./getopt_test -t 
./getopt_test: option requires an argument -- 't'
5:30:31 wing@ubuntu msg ./getopt_test -t 100                                           1 ↵
BBBBBBBB
n = 100
```
		
			