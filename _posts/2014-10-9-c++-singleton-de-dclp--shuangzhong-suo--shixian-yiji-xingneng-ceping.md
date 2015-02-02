---
layout: post
title: C++Singleton的DCLP（双重锁）实现以及性能测评
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
 
  `本文系原创，转载请注明：`<a title="http://www.cnblogs.com/inevermore/p/4014577.html" href="http://www.cnblogs.com/inevermore/p/4014577.html">`http://www.cnblogs.com/inevermore/p/4014577.html`</a>
   
  根据维基百科，对单例模式的描述是：
     确保一个类只有一个实例，并提供对该实例的全局访问。
   从这段话，我们可以得出单例模式的最重要特点：
     一个类最多只有一个对象
    
  
###单线程环境
   
  对于一个普通的类，我们可以任意的生成对象，所以我们为了避免生成太多的类，需要将类的构造函数设置为私有。
  所以我们写出第一步：
  

```C++
class Singleton
{
public:
    
private:
    Singleton() { }
};
```
		



此时在main中就无法直接生成对象：




```C++
Singleton s; //ERROR
```
		



那么我们想要获取实例，只能借助于类内部的函数，于是我们添加一个内部的函数，`而且必须是static函数（思考为什么）：`




```C++
class Singleton
{
public:
    static Singleton *getInstance()
    {
        return new Singleton;
    }
private:
    Singleton() { }
};
```
		
OK，我们可以用这个函数生成对象了，但是每次都去new，无法保证唯一性，于是我们将对象保存在一个static指针内，然后每次获取对象时，先检查该指针是否为空：



```C++
class Singleton
{
public:
    static Singleton *getInstance()
    {
        if(pInstance_ == NULL) //线程的切换
        {
            ::sleep(1);
            pInstance_ = new Singleton;
        }
            
        return pInstance_;
    }
private:
    Singleton() { }

    static Singleton *pInstance_;
};

Singleton *Singleton::pInstance_ = NULL;
```
		



我们在main中测试：




```C++
cout << Singleton::getInstance() << endl;
cout << Singleton::getInstance() << endl;
```
		

可以看到生成了相同的对象，单例模式编写初步成功。


 



###多线程环境下的考虑


 


但是目前的代码就真的没问题了吗？


我写出了以下的测试：




```C++
class TestThread : public Thread
{
public:
    void run()
    {
        cout << Singleton::getInstance() << endl;
        cout << Singleton::getInstance() << endl;
    }
};

int main(int argc, char const *argv[])
{
    //测试证明了多线程下本代码存在竞争问题

    TestThread threads[12];
    for(int ix = 0; ix != 12; ++ix)
    {
        threads[ix].start();
    }

    for(int ix = 0; ix != 12; ++ix)
    {
        threads[ix].join();
    }
    return 0;
}
```
		



 


这里注意，为了达到效果，我特意做了如下改动：




```C++
if(pInstance_ == NULL) //线程的切换
{
     ::sleep(1);
     pInstance_ = new Singleton;
}
```
		

这样`故意造成线程的切换`。


打印结果如下：




```C++
0xb1300468
0xb1300498
0x9f88728
0xb1300498
0xb1300478
0xb1300498
0xb1100488
0xb1300498
0xb1300488
0xb1300498
0xb1300498
0xb1300498
0x9f88738
0xb1300498
0x9f88748
0xb1300498
0xb1100478
0xb1300498
0xb1100498
0xb1300498
0xb1100468
0xb1300498
0xb11004a8
0xb11004a8
```
		



 


很显然，我们的代码在多线程下经不起推敲。


怎么办？加锁！ 于是我们再度改进：




```C++
class Singleton
{
public:
    static Singleton *getInstance()
    {
        mutex_.lock();
        if(pInstance_ == NULL) //线程的切换
            pInstance_ = new Singleton;
        mutex_.unlock();
        return pInstance_;
    }
private:
    Singleton() { }

    static Singleton *pInstance_;
    static MutexLock mutex_;
};

Singleton *Singleton::pInstance_ = NULL;
MutexLock Singleton::mutex_;
```
		

此时测试，无问题。


但是，`互斥锁会极大的降低系统的并发能力，因为每次调用都要加锁，等于一群人过独木桥`。


我写了一份测试如下：




```C++
class TestThread : public Thread
{
public:
    void run()
    {
        const int kCount = 1000 * 1000;
        for(int ix = 0; ix != kCount; ++ix)
        {
            Singleton::getInstance();
        }
    }
};

int main(int argc, char const *argv[])
{
    //Singleton s; ERROR

    int64_t startTime = getUTime();

    const int KSize = 100;
    TestThread threads[KSize];
    for(int ix = 0; ix != KSize; ++ix)
    {
        threads[ix].start();
    }

    for(int ix = 0; ix != KSize; ++ix)
    {
        threads[ix].join();
    }

    int64_t endTime = getUTime();

    int64_t diffTime = endTime - startTime;
    cout << "cost : " << diffTime / 1000 << " ms" << endl;

    return 0;
}
```
		



开了100个线程，每个调用1M次getInstance，其中getUtime的定义如下：




```C++
int64_t getUTime()
{
    struct timeval tv;
    ::memset(&tv, 0, sizeof tv);
    if(gettimeofday(&tv, NULL) == -1)
    {
        perror("gettimeofday");
        exit(EXIT_FAILURE);
    }
    int64_t current = tv.tv_usec;
    current += tv.tv_sec * 1000 * 1000;
    return current;
}
```
		

运行结果为：




```C++
cost : 6914 ms
```
		



 


 



###采用双重锁模式


 


上面的测试，我们还无法看出性能问题，我再次改进代码：




```C++
class Singleton
{
public:
    static Singleton *getInstance()
    {
        if(pInstance_ == NULL)
        {
            mutex_.lock();
            if(pInstance_ == NULL) //线程的切换
                pInstance_ = new Singleton;
            mutex_.unlock();
        }
        
        return pInstance_;
    }
private:
    Singleton() { }

    static Singleton *pInstance_;
    static MutexLock mutex_;
};

Singleton *Singleton::pInstance_ = NULL;
MutexLock Singleton::mutex_;
```
		



可以看到，我在getInstance中采用了两重检查模式，这段代码的优点体现在哪里？



  内部采用互斥锁，代码无论如何是可靠的




  new出第一个实例后，`后面每个线程访问到最外面的if判断就直接返回了`，没有加锁的开销



我再次运行测试，（测试代码不变），结果如下：




```C++
cost : 438 ms
```
		



啊哈，十几倍的性能差距，可见我们的改进是有效的，`仅仅三行代码，却带来了十几倍的效率提升！`


 



###尾声


 


上面这种编写方式成为DCLP（Double-Check-Locking-Pattern）模式，这种方式一度被认为是绝对正确的，但是后来有人指出这种方式在某些情况下也会乱序执行，可以参考Scott的[C++ and the Perils of Double-Checked Locking - Scott Meyer](http://www.aristeia.com/Papers/DDJ_Jul_Aug_2004_revised.pdf)

			