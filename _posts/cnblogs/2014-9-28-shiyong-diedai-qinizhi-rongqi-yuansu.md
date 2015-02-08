---
layout: post
title: 使用迭代器逆置容器元素
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
 
  代码如下：
   
  

```C++
template <typename It>
void reverse(It begin, It end)
{
    while(begin != end)
    {
        --end;
        if(begin != end)
            std::swap(*begin++, *end);
    }
}
```
		



 


 


注意几点：


1.不能一开始就--end，原因是[begin, end)是左闭右开区间，如果begin和end相等，--end则破坏了区间，不是每个迭代器都支持< >操作。


2.在循环内部，不能直接begin++,end--，原因是防止两个相邻的元素。


综合以上原因，只能在循环体内先--end，然后swap的时候begin++；

			