---
layout: post
title: 借助backtrace和demangle实现异常类Exception
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
C++的异常类是没有栈痕迹的，如果需要获取栈痕迹，需要使用以下函数：
  

```C++
#include <execinfo.h>

int backtrace(void **buffer, int size);

char **backtrace_symbols(void *const *buffer, int size);

void backtrace_symbols_fd(void *const *buffer, int size, int fd);
```
		

backtrace将当前程序的调用信息存储在buffer中，backtrace_symbols则是将buffer翻译为字符串。后者用到了malloc，所以需要手工释放内存。


man手册中提供了如下的代码：




```Python
#include <execinfo.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void
myfunc3(void)
{
   int j, nptrs;
#define SIZE 100
   void *buffer[100];
   char **strings;

   nptrs = backtrace(buffer, SIZE);
   printf("backtrace() returned %d addresses\n", nptrs);

   /* The call backtrace_symbols_fd(buffer, nptrs, STDOUT_FILENO)
      would produce similar output to the following: */

   strings = backtrace_symbols(buffer, nptrs);
   if (strings == NULL) {
       perror("backtrace_symbols");
       exit(EXIT_FAILURE);
   }

   for (j = 0; j < nptrs; j++)
       printf("%s\n", strings[j]);

   free(strings);
}

static void   /* "static" means don't export the symbol... */
myfunc2(void)
{
   myfunc3();
}

void
myfunc(int ncalls)
{
   if (ncalls > 1)
       myfunc(ncalls - 1);
   else
       myfunc2();
}

int
main(int argc, char *argv[])
{
   if (argc != 2) {
       fprintf(stderr, "%s num-calls\n", argv[0]);
       exit(EXIT_FAILURE);
   }

   myfunc(atoi(argv[1]));
   exit(EXIT_SUCCESS);
}
```
		

编译并执行：




```C++
$  cc -rdynamic prog.c -o prog
$ ./prog 3
```
		



输出如下：




```C++
backtrace() returned 8 addresses
./prog(myfunc3+0x1f) [0x8048783]
./prog() [0x8048810]
./prog(myfunc+0x21) [0x8048833]
./prog(myfunc+0x1a) [0x804882c]
./prog(myfunc+0x1a) [0x804882c]
./prog(main+0x52) [0x8048887]
/lib/i386-linux-gnu/libc.so.6(__libc_start_main+0xf3) [0xb76174d3]
./prog() [0x80486d1]
```
		



 


因此我写出以下的异常类，注意上面的打印结果经过了名字改编，所以我们使用abi::__cxa_demangle将名字还原出来。


Exception.h




```Python
#ifndef EXCEPTION_H_
#define EXCEPTION_H_

#include <string>
#include <exception>

class Exception : public std::exception
{
public:
    explicit Exception(const char* what);
    explicit Exception(const std::string& what);
    virtual ~Exception() throw();
    virtual const char* what() const throw();
    const char* stackTrace() const throw();

private:
    void fillStackTrace();  //填充栈痕迹
    std::string demangle(const char* symbol); //反名字改编

    std::string message_;   //异常信息
    std::string stack_;     //栈trace
};


#endif  // EXCEPTION_H_
```
		

Exception.cpp




```C++
#include "Exception.h"
#include <cxxabi.h>
#include <execinfo.h>
#include <stdlib.h>
#include <stdio.h>

using namespace std;

Exception::Exception(const char* msg)
  : message_(msg)
{
    fillStackTrace();
}

Exception::Exception(const string& msg)
  : message_(msg)
{
    fillStackTrace();
}

Exception::~Exception() throw ()
{
}

const char* Exception::what() const throw()
{
    return message_.c_str();
}

const char* Exception::stackTrace() const throw()
{
    return stack_.c_str();
}

//填充栈痕迹
void Exception::fillStackTrace()
{
    const int len = 200;
    void* buffer[len];
    int nptrs = ::backtrace(buffer, len); //列出当前函数调用关系
    //将从backtrace函数获取的信息转化为一个字符串数组
    char** strings = ::backtrace_symbols(buffer, nptrs); 

    if (strings)
    {
        for (int i = 0; i < nptrs; ++i)
        {
          // TODO demangle funcion name with abi::__cxa_demangle
          //strings[i]代表某一层的调用痕迹
          stack_.append(demangle(strings[i]));
          stack_.push_back('\n');
        }
        free(strings);
    }
}

//反名字改编
string Exception::demangle(const char* symbol)
{
    size_t size;
    int status;
    char temp[128];
    char* demangled;
    //first, try to demangle a c++ name
    if (1 == sscanf(symbol, "%*[^(]%*[^_]%127[^)+]", temp)) {
        if (NULL != (demangled = abi::__cxa_demangle(temp, NULL, &size, &status))) {
          string result(demangled);
          free(demangled);
          return result;
        }
    }
    //if that didn't work, try to get a regular c symbol
    if (1 == sscanf(symbol, "%127s", temp)) {
        return temp;
    }

    //if all else fails, just return the symbol
    return symbol;
}
```
		



测试代码如下：




```C++
#include "Exception.h"
#include <stdio.h>
using namespace std;

class Bar
{
 public:
  void test()
  {
    throw Exception("oops");
  }
};

void foo()
{
  Bar b;
  b.test();
}

int main()
{
  try
  {
    foo();
  }
  catch (const Exception& ex)
  {
    printf("reason: %s\n", ex.what());
    printf("stack trace: %s\n", ex.stackTrace());
  }
}
```
		



打印结果如下：




```C++
reason: oops
stack trace: Exception::fillStackTrace()
Exception::Exception(char const*)
Bar::test()
foo()
./a.out(main+0xf)
/lib/i386-linux-gnu/libc.so.6(__libc_start_main+0xf3)
./a.out()
```
		



注意编译的时候，加上-rdynamic选项


 


有了这个类，我们可以在程序中这样处理异常：




```C++
try
    {
        //
    }
    catch (const Exception& ex)
    {
        fprintf(stderr, "reason: %s\n", ex.what());
        fprintf(stderr, "stack trace: %s\n", ex.stackTrace());
        abort();
    }
    catch (const std::exception& ex)
    {
        fprintf(stderr, "reason: %s\n", ex.what());
        abort();
    }
    catch (...)
    {
        fprintf(stderr, "unknown exception caught \n");
    throw; // rethrow
    }
```
		
			