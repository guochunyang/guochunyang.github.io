---
layout: post
title: 模板系列（一）模板的模板参数
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
在之前，我们写过类似的stack



```C++
template <typename T, typename Alloc = std::vector<T> >
class Stack
{
public:
    void push(const T &);
    void pop();
    T top() const;
    bool empty() const;
private:
    Alloc cont_;
};
```
		
那么我们使用的时候，需要这样写



```C++
Stack<string, list<string> > st;
```
		
我们看到，string这个类型参数出现了两次，那么可不可以消除呢？

显然我们的目的是只指定容器的类型，而不包括元素的类型，这就需要借助模板的模板参数，来帮助我们写出一下代码：



```C++
Stack<string, list> st;
```
		
那么如何使用模板的模板参数呢？

我们需要这样定义模板参数的第二项修改为模板



```C++
template <typename T, 
          template <typename ELEM> class Alloc = std::vector>
class Stack
```
		
我们回忆一下，我们没接触泛型时，写的是普通的stack



```C++
class Stack;
```
		
当我们开始使用泛型时，就把这个类名加上模板参数。



```C++
template <typename T, typename Alloc = std::vector<T> >
class Stack;
```
		
上面两项叫做类的模板参数。现在我们需要把第二个模板参数改为泛型的，于是，我们给它加上模板参数：



```C++
template <typename T, 
          template <typename ELEM> </pre>
<pre>          typename Alloc = std::vector>
class Stack;
```
		
但是上面的写法有一处错误，上面的Alloc必须是一个类，所以我们只能用class作为关键字，于是改为：



```C++
template <typename T, 
          template <typename ELEM> </pre>
<pre>          class Alloc = std::vector>
class Stack;
```
		
我们写出完整的一份实现：



```C++
template <typename T, 
          template <typename ELEM> </pre>
<pre>          class Alloc = std::vector>
class Stack
{
public:
    void push(const T &);
    void pop();
    T top() const
    {
        return cont_.back();
    }
    bool empty() const
    {
        return cont_.empty();
    }
private:
    Alloc<T> cont_;
};

template <typename T, template <typename> class Alloc>
void Stack<T, Alloc>::push(const T &val)
{
    cont_.push_back(val);
}

template <typename T, template <typename> class Alloc>
void Stack<T, Alloc>::pop()
{
    cont_.pop_back();
}
```
		
当我们使用：



```C++
Stack<string, list> st;
```
		
却不幸发现存在编译错误，有一条是：


expected a template of type &lsquo;template<class ELEM> class Alloc&rsquo;, got &lsquo;template<class _Tp, class _Alloc> class std::list


这是因为，无论是vector还是list都有两个模板参数，于是无法与Alloc这个模板参数匹配。

我们修正为：



```C++
template <typename T, 
          template <typename ELEM, typename Alloc = std::allocator<ELEM> > 
          class Container = std::vector>
class Stack;
```
		
然后给出完整实现：



```C++
template <typename T, 
          template <typename ELEM, typename Alloc = std::allocator<ELEM> > 
          class Container = std::vector>
class Stack
{
public:
    void push(const T &);
    void pop();
    T top() const
    {
        return cont_.back();
    }
    bool empty() const
    {
        return cont_.empty();
    }
private:
    Container<T> cont_;
};

template <typename T, template <typename, typename> class Container>
void Stack<T, Container>::push(const T &val)
{
    cont_.push_back(val);
}

template <typename T, template <typename, typename> class Container>
void Stack<T, Container>::pop()
{
    cont_.pop_back();
}
```
		
测试代码：



```C++
Stack<string, list> st;
st.push("foo");
st.pop();
```
		
完毕！

			