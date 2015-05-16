---
layout: post
title: 探究加法操作的原子性
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---

###加法在多线程下是否可靠
  

```C++
#include <iostream>
#include <string>
#include <vector>
#include <thread>
#include <stdio.h>
#include <memory>
using namespace std; 

int g_count = 0; 

int main(int argc, const char *argv[])
{
    vector<unique_ptr<thread>> ths;
    for (int i = 0; i < 100; i++) {
        ths.push_back(unique_ptr<thread>(new thread([]()
                {
                    for (int i = 0; i < 1000000; i++) {
                        ++g_count;
                    }
                })));
    } 

    for(auto &f : ths)
    {
        f->join();
    } 

    printf("count = %d\n", g_count);
    return 0;
}
```
		

这段代码使用了C++11的thread模块以及lambda匿名函数特性。每个线程都对全局变量count进行加一，一共100个线程，每个线程执行1000000，那么最后的结果应该是100 * 1000000 = 100000000，一亿。 我们查看下实际的结果： 




```C++
➜  test  g++ 1.cc -std=c++11 -lpthread
➜  test  ./a.out
count = 64526893
➜  test  ./a.out
count = 88669187
➜  test  ./a.out
count = 60832281
➜  test  ./a.out
count = 82043096
➜  test  ./a.out
count = 65450522
➜  test
```
		


  <br />结果和我们预期的不太一样，结果低于 10000 0000.那么这是为什么呢？ 



###加法的原理




```C++
mov eax, DWORD PTR [test]  
inc eax  
mov DWORD PTR [text], eax
```
		


  <br />这是一段VS下的汇编（本人不会写汇编，这段代码由他人提供，虽然不是Linux平台，但是原理类似），从这段代码，我们可以看出。加一操作是分为三个步骤： 



      1.从内存中取数据到寄存器  <br />    2.对寄存器执行+1  <br />    3.将结果存回内存



但是，上面的这三个指令不具有原子性，也就是说，这三个操作有可能被打断，而不是一气呵成的。 如果两个线程同时执行+1，那么这两个线程可能交替执行，过程如下： 



  1.线程A从内存中取到数据，此时为33
    <br />2.线程B取到也为33.

    <br />3.A对数据执行加法，为34

    <br />4.A将结果返回内存，保存在34

    <br />5.B对数据执行加法，为34，

    <br />6.B将结果返回内存，为34.

    <br />



一个值为33的变量，相加2次，结果居然是34.这个错误的结果正是加法操作的费原子性导致的。 


那么解决方案在哪里？ 



###加锁保护加法操作




```C++
#include <iostream>
#include <string>
#include <vector>
#include <thread>
#include <stdio.h>
#include <mutex>
#include <memory>
using namespace std; 

int g_count = 0;
std::mutex g_lock; 

int main(int argc, const char *argv[])
{
    vector<unique_ptr<thread>> ths;
    for (int i = 0; i < 100; i++) {
        ths.push_back(unique_ptr<thread>(new thread([]()
                {
                    for (int i = 0; i < 1000000; i++) {
                        g_lock.lock();
                        ++g_count;
                        g_lock.unlock();
                    }
                })));
    } 

    for(auto &f : ths)
    {
        f->join();
    } 

    printf("count = %d\n", g_count);
    return 0;
}
```
		

 


然后查看运行结果： 


➜  test  ./a.out
  <br />count = 100000000

  <br />➜  test  ./a.out

  <br />count = 100000000

  <br />➜  test  ./a.out

  <br />count = 100000000

  <br />➜  test  ./a.out

  <br />count = 100000000

  <br />➜  test

  <br />我们看到使用加锁，保证了结果的正确性，因此加锁是种可行的解决方案。 



###使用CAS原子操作
      <br />


那么好的解决方案是什么？我们采用CAS。
  <br />CAS是一组原子操作，他可以防止运行这些指令时，CPU不被打断，从而保证操作的原子性。

  <br />有关CAS，可以参考http://en.wikipedia.org/wiki/Compare-and-swap 


gcc提供了一组原子操作，参考http://gcc.gnu.org/onlinedocs/gcc-4.1.2/gcc/Atomic-Builtins.html 


这里我们使用_syncfetchandadd，代码如下： 




```C++
#include <iostream>
#include <string>
#include <vector>
#include <thread>
#include <stdio.h>
#include <memory>
using namespace std; 

int g_count = 0; 

int main(int argc, const char *argv[])
{
    vector<unique_ptr<thread>> ths;
    for (int i = 0; i < 100; i++) {
        ths.push_back(unique_ptr<thread>(new thread([]()
                {
                    for (int i = 0; i < 1000000; i++) {
                        //++g_count;
                        __sync_fetch_and_add(&g_count, 1);
                    }
                })));
    } 

    for(auto &f : ths)
    {
        f->join();
    } 

    printf("count = %d\n", g_count);
    return 0;
}
```
		

 


 


查看运行结果，依然是正确的。 



###比较效率


运行错误的代码，测评如下： 


➜  test  time ./a.out
  <br />count = 75377651

  <br />./a.out  0.24s user 0.00s system 99% cpu 0.246 total

  <br />➜  test  time ./a.out

  <br />count = 63644431

  <br />./a.out  0.25s user 0.00s system 99% cpu 0.254 total

  <br />➜  test  time ./a.out

  <br />count = 23350388

  <br />./a.out  0.25s user 0.00s system 99% cpu 0.257 total

  <br />➜  test

  <br />加锁的代码如下： 


➜  test  g++ 1.cc -std=c++11 -lpthread
  <br />➜  test  time ./a.out

  <br />count = 100000000

  <br />./a.out  2.71s user 0.01s system 99% cpu 2.723 total

  <br />➜  test  time ./a.out

  <br />count = 100000000

  <br />./a.out  2.66s user 0.01s system 99% cpu 2.676 total

  <br />➜  test  time ./a.out

  <br />count = 100000000

  <br />./a.out  2.69s user 0.01s system 99% cpu 2.699 total

  <br />➜  test

  <br />使用原子操作的代码： 


➜  test  g++ 3.cc -std=c++11 -lpthread
  <br />➜  test  time ./a.out

  <br />count = 100000000

  <br />./a.out  0.67s user 0.00s system 99% cpu 0.676 total

  <br />➜  test  time ./a.out

  <br />count = 100000000

  <br />./a.out  0.65s user 0.00s system 99% cpu 0.655 total

  <br />➜  test  time ./a.out

  <br />count = 100000000

  <br />./a.out  0.65s user 0.01s system 99% cpu 0.661 total

  <br />➜  test

  <br />可以看到，使用原子操作，即保证了效率，又保证了安全性。

  <br />加锁之所以效率低，原因在于它的粒度太粗，当一个线程加锁失败，让出CPU，此时CPU的大部分调度都是无用功，因为其它线程无法继续向下执行。

  <br />而原子操作，直接避免了让出CPU，因为它通常都比较短，远远低于一个线程分配的时间片，所以对系统没有负面影响。 



###C++11提供的原子操作




```C++
#include <iostream>
#include <string>
#include <vector>
#include <thread>
#include <stdio.h>
#include <memory>
#include <atomic>
using namespace std; 

atomic_int g_count; 

int main(int argc, const char *argv[])
{
    vector<unique_ptr<thread>> ths;
    for (int i = 0; i < 100; i++) {
        ths.push_back(unique_ptr<thread>(new thread([]()
                {
                    for (int i = 0; i < 1000000; i++) {
                        //++g_count;
                        ++g_count;
                    }
                })));
    } 

    for(auto &f : ths)
    {
        f->join();
    } 

    //printf("count = %d\n", g_count);
    printf("count = %d\n", g_count.load());
    return 0;
}
```
		

 


结果： 


➜  test  g++ 4.cc -std=c++11 -lpthread
  <br />➜  test  ./a.out

  <br />count = 100000000

  <br />➜  test  ./a.out

  <br />count = 100000000

  <br />➜  test  time ./a.out

  <br />count = 100000000

  <br />./a.out  0.88s user 0.01s system 99% cpu 0.886 total

  <br />➜  test  time ./a.out

  <br />count = 100000000

  <br />./a.out  0.88s user 0.01s system 99% cpu 0.891 total

  <br />➜  test  time ./a.out

  <br />count = 100000000

  <br />./a.out  0.88s user 0.01s system 99% cpu 0.893 total

			