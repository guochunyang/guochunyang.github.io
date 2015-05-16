---
layout: post
title: 分析一个C语言程序生成的汇编代码-《Linux内核分析》Week1作业
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
###署名信息

	郭春阳 原创作品转载请注明出处 ：《Linux内核分析》MOOC课程 http://mooc.study.163.com/course/USTC-1000029000 

###C源码


这里为了防止重复，修改了部分源码

```C
int g(int x)
{
  return x + 99;
}

int f(int x)
{
  return g(x);
}

int main(void)
{
  return f(22) + 36;
}
```

运行 `gcc -S -o foo.s -m32 foo.c`后，生成的汇编代码为：

```asm
g:
	pushl	%ebp
	movl	%esp, %ebp
	movl	8(%ebp), %eax
	addl	$99, %eax
	popl	%ebp
	ret
f:
	pushl	%ebp
	movl	%esp, %ebp
	pushl	8(%ebp)
	call	g
	addl	$4, %esp
	leave
	ret
main:
	pushl	%ebp
	movl	%esp, %ebp
	pushl	$22
	call	f
	addl	$4, %esp
	addl	$36, %eax
	leave
	ret
```

###分析代码运行过程

首先进入main函数：

	`pushl	 %ebp, movl %esp, %ebp`两行代码，将ebp压入栈，  
	然后esp的值赋给ebp，这里esp的值先减4，ebp指向与esp相同的位置  
	以上指令相当于enter，进入了一个新的函数调用栈  
	将22压入栈（这里其实是定义接下来调用的f的形参int x，并赋值）  
	调用f，这里先保存eip（PC计数器）的值入栈，esp的值-4，然后将f的地址赋给eip  
	
进入f函数：

	执行enter的两条指令，进入新的函数调用栈（同上）  
	将ebp向前8个字节位置的值（main中压栈的22）压栈，esp-4  
	这里是定义g的形参并赋值  
	调用g，eip先压栈，然后eip变为g的地址

进入g函数：

	首先执行enter的两条指令，进入新的调用栈（同上）  
	`movl	8(%ebp), %eax` 首先寻址，将f中压栈的数（也就是g的形参x），赋给eax  
	`addl	$99, %eax` eax中得值+99，对应C语句 x + 99  
	`popl	%ebp` 这里因为g中没有开辟新的栈空间（esp的值没有变化），所以这句和leave语句等价  
	执行完，这一句，ebp恢复到之前的值（f中得状态）  
	执行ret，其实是执行popl eip，将栈中保存的eip的值，赋给eip  
	此时ebp、esp和eip均恢复到调用f中得状态
	退出g，返回值保存在eax中  

回到f函数：

	`addl    $4, %esp`  esp+4，因为前面进行了pushl开辟新的变量，所以这里需要丢弃栈顶  
	之后执行leave 和 ret，恢复esp、ebp、eip的值，同上
	f仅仅返回g的返回值，这里不改动eax，里面保存着g的返回值
	
回到main函数：

	`addl    $4, %esp`丢弃栈顶，原因是上面进行push，定义了新的变量  
	` addl    $36, %eax` f(22)的返回值+36，相当于C中得 f(22) + 36
	main函数leave ret 返回上一层 
	gcc中应该为__libc_start_main，它可以接收到main的返回值，在eax中
	
###注意点

	1.C中的函数，是先定义形参并赋值，然后才进入该函数代码  
	2.函数调用中占用的栈空间，不仅仅包括局部变量，还包括22这种常量  
	3.每次函数调用，都执行enter，这条指令的效果相当于开辟了一个新的空栈  
	4.call调用函数时，eip保存的是call的下一条指令  
	
###博客截图

	实验环境是在Ubuntu 14.10 32bit，在mac OS X上远程登录  
![](http://images.cnitblog.com/blog2015/669654/201503/080005124456292.png)


	
###对计算机工作流程的认识  

	图灵机可以执行某些特定操作 
	通用图灵机可以执行传给它的指令  
	现在计算机采用冯诺依曼体系结构，将程序和数据均存储在内存中，只需要程序员编写计算机可以识别的代码即可，计算机会将这些代码编译为可执行的指令。
			