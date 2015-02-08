---
layout: post
title: 标准库priority_queue的一种实现
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
优先级队列相对于普通队列，提供了插队功能，每次最先出队的不是最先入队的元素，而是优先级最高的元素。
  它的实现采用了标准库提供的heap算法。该系列算法一共提供了四个函数。使用方式如下：
  首先，建立一个容器，放入元素：
  

```C++
vector<int> coll;
insertNums(coll, 3, 7);
insertNums(coll, 5, 9);
insertNums(coll, 1, 4);

printElems(coll, "all elements: ");
```
		

打印结果为：
  

```C++
all elements: 
3 4 5 6 7 5 6 7 8 9 1 2 3 4
```
		



然后我们调用make_heap，这个算法把[beg, end)内的元素建立成堆。




```C++
make_heap(coll.begin(), coll.end());

printElems(coll, "after make_heap: ");
```
		

打印结果：
  

```C++
after make_heap: 
9 8 6 7 7 5 5 3 6 4 1 2 3 4
```
		



然后我们调用pop_heap，这个算法必须保证[beg, end)已经是一个heap，然后它将堆顶的元素（其实是begin指向的元素）放到最后，再把[begin. end-1)内的元素重新调整为heap




```C++
pop_heap(coll.begin(), coll.end());
coll.pop_back();
printElems(coll, "after pop_heap: ");
```
		





打印结果为：




```C++
after pop_heap: 
8 7 6 7 4 5 5 3 6 4 1 2 3
```
		



接下来我们调用push_heap，该算法必须保证[beg, end-1)已经是一个heap，然后将整个[beg, end)调整为heap




```C++
coll.push_back(17);
push_heap(coll.begin(), coll.end());

printElems(coll, "after push_heap: ");
```
		

打印结果为：




```C++
after push_heap: 
17 7 8 7 4 5 6 3 6 4 1 2 3 5
```
		



最后我们使用sort_heap将[beg, end)由heap转化为有序序列，所以，前提是[beg, end)已经是一个heap




```C++
sort_heap(coll.begin(), coll.end());
printElems(coll, "after sort_heap: ");
```
		

打印结果为：




```C++
after sort_heap: 
1 2 3 3 4 4 5 5 6 6 7 7 8 17
```
		



完整的测试代码如下：




```C++
#include <iostream>
#include <string>
#include <vector>
#include <algorithm>
using namespace std;

template <typename T>
void insertNums(T &t, int beg, int end)
{
    while(beg <= end)
    {
        t.insert(t.end(), beg);
        ++beg;
    }    
}

template <typename T>
void printElems(const T &t, const string &s = "")
{
    cout << s << endl;
    for(typename T::const_iterator it = t.begin();
        it != t.end();
        ++it)
    {
        cout << *it << " ";
    }
    cout << endl;
}


int main(int argc, char const *argv[])
{
    vector<int> coll;
    insertNums(coll, 3, 7);
    insertNums(coll, 5, 9);
    insertNums(coll, 1, 4);

    printElems(coll, "all elements: ");

    //在这个范围内构造heap
    make_heap(coll.begin(), coll.end());

    printElems(coll, "after make_heap: ");

    //将堆首放到最后一个位置，其余位置调整成堆
    pop_heap(coll.begin(), coll.end());
    coll.pop_back();
    printElems(coll, "after pop_heap: ");

    coll.push_back(17);
    push_heap(coll.begin(), coll.end());

    printElems(coll, "after push_heap: ");

    sort_heap(coll.begin(), coll.end());
    printElems(coll, "after sort_heap: ");

    return 0;
}
```
		

 


根据以上的算法，我们来实现标准库的优先级队列priority_queue，代码如下：




```Python
#ifndef PRIORITY_QUEUE_HPP
#define PRIORITY_QUEUE_HPP

#include <vector>
#include <algorithm>
#include <functional>

template <typename T, 
          typename Container = std::vector<T>, 
          typename Compare = std::less<typename Container::value_type> >
class PriorityQueue
{
public:
    typedef typename Container::value_type value_type; //不用T
    typedef typename Container::size_type size_type;
    typedef Container container_type;
    typedef value_type &reference;
    typedef const value_type &const_reference;


    PriorityQueue(const Compare& comp = Compare(),
                  const Container& ctnr = Container());
    template <class InputIterator>
    PriorityQueue (InputIterator first, InputIterator last,
                   const Compare& comp = Compare(),
                   const Container& ctnr = Container());
    void push(const value_type &val)
    {
        cont_.push_back(val);
        //调整最后一个元素入堆
        std::push_heap(cont_.begin(), cont_.end(), comp_); 
    }

    void pop()
    {
        //第一个元素移出堆，放在最后
        std::pop_heap(cont_.begin(), cont_.end(), comp_);
        cont_.pop_back();
    }

    bool empty() const { return cont_.empty(); }
    size_type size() const { return cont_.size(); }
    const_reference top() const { return cont_.front(); }

private:
    Compare comp_; //比较规则
    Container cont_; //内部容器
};

template <typename T, typename Container, typename Compare>
PriorityQueue<T, Container, Compare>::PriorityQueue(const Compare& comp,
                                                    const Container& ctnr)
    :comp_(comp), cont_(ctnr)
{
    std::make_heap(cont_.begin(), cont_.end(), comp_); //建堆
}

template <typename T, typename Container, typename Compare>
template <class InputIterator>
PriorityQueue<T, Container, Compare>::PriorityQueue (InputIterator first, 
                                                     InputIterator last,
                                                     const Compare& comp,
                                                     const Container& ctnr)
    :comp_(comp), cont_(ctnr)
{
    cont_.insert(cont_.end(), first, last);
    std::make_heap(cont_.begin(), cont_.end(), comp_);
}

#endif //PRIORITY_QUEUE_HPP
```
		



我们注意到：



  1.优先级队列内部保存了排序规则，这与map和set是一致的。


  2.前面我们提到heap算法除了make_heap之外，都必须保证之前是一个建好的heap，这里我们在构造函数中调用make_heap，保证了后面的各种heap算法都是合法的。


  3.还有一点，如果T与容器的类型不一致，例如PriorityQueue<float, vector<int> >，那么我们的value_type优先采用int，毕竟我们操作的对象是容器。



测试代码如下：




```C++
#include "PriorityQueue.hpp"
#include <iostream>
using namespace std;

int main(int argc, char const *argv[])
{
    PriorityQueue<float> q;
    q.push(66.6);
    q.push(22.3);
    q.push(44.4);


    cout << q.top() << endl;
    q.pop();
    cout << q.top() << endl;
    q.pop();

    q.push(11.1);
    q.push(55.5);
    q.push(33.3);
    q.pop();

    while(!q.empty())
    {
        cout << q.top() << " ";
        q.pop();
    }
    cout << endl;


    return 0;
}
```
		
			