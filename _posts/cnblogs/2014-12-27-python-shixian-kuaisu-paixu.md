---
layout: post
title: Python实现快速排序
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
这里采用的是算法导论的划分方式：
  

```Python
import random

def partition(array, left, right):
    pivot = array[left]
    i = left
    #j left +1 -> right
    for j in range(left + 1, right + 1):
        if array[j] < pivot:
            i += 1
            temp = array[i]
            array[i] = array[j]
            array[j] = temp
    temp = array[left]
    array[left] = array[i]
    array[i] = temp
    return i

def quickSort(array, left, right):
    if left < right:
        pos = partition(array, left, right)
        quickSort(array, left, pos-1)
        quickSort(array, pos+1, right)

if __name__ == '__main__':
    arr = []
    for i in range(40):
        arr.append(random.randrange(10,100))
    print arr
    quickSort(arr, 0, len(arr)-1)
    print arr
```
		

 


运行结果为：




```C++
~/Documents/py python 5.py
[44, 46, 89, 97, 89, 68, 36, 28, 43, 74, 19, 62, 62, 53, 30, 30, 32, 98, 62, 25, 63, 37, 94, 21, 46, 93, 63, 80, 76, 62, 57, 24, 53, 49, 90, 67, 48, 75, 96, 75]
[19, 21, 24, 25, 28, 30, 30, 32, 36, 37, 43, 44, 46, 46, 48, 49, 53, 53, 57, 62, 62, 62, 62, 63, 63, 67, 68, 74, 75, 75, 76, 80, 89, 89, 90, 93, 94, 96, 97, 98]
```
		

这里代码的风格仍侧重于C语言。


我将两个数字交换的代码,写成python风格:




```Python
def partition(array, left, right):
    pivot = array[left]
    i = left
    #j left +1 -> right
    for j in range(left + 1, right + 1):
        if array[j] < pivot:
            i += 1
            array[i], array[j] = array[j], array[i]
    array[left], array[i] = array[i], array[left]
    return i
```
		


这样就简洁了很多
			