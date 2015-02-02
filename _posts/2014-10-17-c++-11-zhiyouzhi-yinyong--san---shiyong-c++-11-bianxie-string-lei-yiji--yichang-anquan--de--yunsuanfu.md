---
layout: post
title: C++11之右值引用（三）：使用C++11编写string类以及“异常安全”的=运算符
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
前面两节，说明了右值引用和它的作用。下面通过一个string类的编写，来说明右值引用的使用。
  相对于C++98，`主要是多了移动构造函数和移动赋值运算符`。
  先给出一个简要的声明：
  

```C++
class String
{
public:
    String();
    String(const char *s); //转化语义
    String(const String &s);
    String(String &&s);
    ~String();

    String &operator=(const String &s);
    String &operator=(String &&s);

    friend ostream &operator<<(ostream &os, const String &s)
    {
        return os << s.data_;
    }
private:
    char *data_;
};
```
		



下面依次实现每个函数。


第一个是默认构造函数：




```Python
String::String()
:data_(new char[1])
{
    *data_ = 0;
    cout << "default" << endl;
}
```
		

 


然后是char*版本的构造函数：




```C++
String::String(const char *s)
:data_(new char[strlen(s) + 1])
{
    ::strcpy(data_, s);
    cout << "char *" << endl;
}
```
		

重点来了，我们提供移动构造函数：




```C++
String::String(String &&s)
:data_(s.data_)
{
    cout << "move construct" << endl;
    s.data_ = NULL; //防止释放data
}
```
		



这里最重要的一点就是要把s的data置为NULL，因为s是个右值，马上就要析构。这样就`成功实现了偷取s的内容`。


析构函数：




```C++
String::~String()
{
    delete[] data_;
}
```
		

下面我们提供赋值运算符，这里注意一点：


一是处理自我赋值，二是要返回自身引用。




```C++
String &String::operator=(const String &s)
{
    if(this != &s)
    {
        delete[] data_;
        data_ = new char[strlen(s.data_) + 1];
        ::strcpy(data_, s.data_);
    }
    return *this;
}

String &String::operator=(String &&s)
{
    if(this != &s)
    {
        cout << "move assignment" << endl;
        delete[] data_;
        data_ = s.data_;
        s.data_ = NULL;
    }
    return *this;
}
```
		



后面的移动构造函数，依然要把s的data置为NULL。


上面两个函数看似正确，但是没有处理发生异常的情况，`如果new时发生异常，但是此时原本的data已经被delete，造成错误`。


如何解决？


我们提供一个swap函数：




```C++
void String::swap(String &s)
{
    std::swap(data_, s.data_);
}
```
		



一种好的处理方案是：




```C++
String &String::operator=(const String &s)
{
    String temp(s);
    swap(temp);

    return *this;
}

String &String::operator=(String &&s)
{
    String temp(s);
    swap(temp);

    return *this;
}
```
		



这样，即使生成temp时发生异常，也对自身没有影响。


注意这里没有处理自我赋值，因为自我赋值发生的情况实际比较少，而之前的代码第一行是delete，则必须处理自我赋值。


上面两个赋值运算符可以直接合为一个：




```C++
String &String::operator=(String s)
{
    swap(s);

    return *this;
}
```
		



事实上，我们在前面也提到过，除了构造函数之外，`X &x和X &&类型的函数，可以合二为一为X x，采用传值`。


这样，我们的最后一个实现，保证了异常安全。


 


测试代码：




```C++
int main(int argc, char const *argv[])
{
    String s("foo");
    String s2(s);
    //String s3(std::move(String("bar")));
    String s3(String("bar")); //编译器优化 直接使用char*
    cout << s3 << endl;

    s3 = s;
    s3 = String("hello");
    cout << s3 << endl;
    s3 = std::move(s2);
    cout << s3 << endl;

    return 0;
}
```
		

注意：




```C++
String s3(String("bar"));
```
		



会被编译器优化为




```C++
String s3(“bar”)
```
		

可以显式使用：




```C++
String s3(std::move(String("bar")));
```
		



 


完毕。

			