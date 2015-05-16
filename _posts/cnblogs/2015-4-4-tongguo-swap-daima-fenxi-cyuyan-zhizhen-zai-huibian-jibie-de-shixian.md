---
layout: post
title: 通过swap代码分析C语言指针在汇编级别的实现
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
我们先用C语言写一个交换两个数的代码：

```C
void swap(int *a, int *b){
    int temp = *a;
    *a = *b;
    *b = temp;
}

int main(void)
{
    int x = 12;
    int y = 34;
    swap(&a, &b);
    return 0;
}
```

我们使用下面的命令进行编译，得到汇编文件：

	gcc -o 1.s -S 1.c -m32
	
查看汇编文件，这里去掉了许多.开头的符号：

```asm
swap:
	pushl	%ebp
	movl	%esp, %ebp //
	subl	$16, %esp
	movl	8(%ebp), %eax // a -> eax
	movl	(%eax), %eax  // *a -> eax
	movl	%eax, -4(%ebp) // *a -> temp
	movl	12(%ebp), %eax // b -> eax
	movl	(%eax), %edx // *b -> edx
	movl	8(%ebp), %eax // a -> eax
	movl	%edx, (%eax) // *b -> *a
	movl	12(%ebp), %eax // b -> eax
	movl	-4(%ebp), %edx // temp -> edx
	movl	%edx, (%eax) // edx -> *b
	leave
	ret
main:
	leal	4(%esp), %ecx
	andl	$-16, %esp
	pushl	-4(%ecx)
	pushl	%ebp
	movl	%esp, %ebp
	pushl	%ecx
	subl	$20, %esp
	movl	%gs:20, %eax
	movl	%eax, -12(%ebp)
	xorl	%eax, %eax
	movl	$12, -20(%ebp) // x
	movl	$34, -16(%ebp) // y
	leal	-16(%ebp), %eax // &y -> eax
	pushl	%eax // &y 入栈
	leal	-20(%ebp), %eax // &x -> ebx
	pushl	%eax // &x 入栈
	call	swap
	addl	$8, %esp
	movl	$0, %eax
	movl	-12(%ebp), %edx
	xorl	%gs:20, %edx
	je	.L4
	call	__stack_chk_fail
	movl	-4(%ebp), %ecx
	leave
	leal	-4(%ecx), %esp
	ret
```

我们先分析main中这几行代码：

```asm
	movl	$12, -20(%ebp) // x
	movl	$34, -16(%ebp) // y
	leal	-16(%ebp), %eax // &y -> eax
	pushl	%eax // &y 入栈
	leal	-20(%ebp), %eax // &x -> ebx
	pushl	%eax // &x 入栈
	call	swap
```

首先前面两行代码分别将12、34压入栈，也就是main中的x和y。  
后面有一句`leal	-16(%ebp), %eax`，leal的意思是将源操作数的地址传给有操作数，所以这句的作用是取y的地址赋给eax。  
下一句将eax也就是y的地址压入栈，这个其实是swap的最后一个形参b。  
后面两句类似，将x的地址压栈，也就是swap的形参a。  

我们看到，函数参数的压栈顺序是从右向左。

然后我们分析swap的代码：

```asm
	movl	8(%ebp), %eax // a -> eax
	movl	(%eax), %eax  // *a -> eax
	movl	%eax, -4(%ebp) // *a -> temp
	movl	12(%ebp), %eax // b -> eax
	movl	(%eax), %edx // *b -> edx
	movl	8(%ebp), %eax // a -> eax
	movl	%edx, (%eax) // *b -> *a
	movl	12(%ebp), %eax // b -> eax
	movl	-4(%ebp), %edx // temp -> edx
	movl	%edx, (%eax) // edx -> *b
```

在这里注意，每当发生函数调用时，先将形参准备好入栈，然后依次是eip、ebp。  
由于栈的地址是由高到低增长，所以，在swap中`12(%ebp)`指的是b，`8(%ebp)`指的是a，`-4(%ebp)`指temp。

所以上面代码执行的步骤就是：

```asm
	movl	8(%ebp), %eax // a -> eax
	movl	(%eax), %eax  // *a -> eax
	movl	%eax, -4(%ebp) // *a -> temp
```

分别是将a赋值给eax，然后对a解引用，赋给eax，此时eax中就是*a，也就是x的值。第三行将x的值赋给temp。

```asm
	movl	12(%ebp), %eax // b -> eax
	movl	(%eax), %edx // *b -> edx
	movl	8(%ebp), %eax // a -> eax
	movl	%edx, (%eax) // *b -> *a
```
将b也就是y的地址赋给eax，然后解引用，y的值赋给edx。然后a也就是x的地址赋给eax，最后一行将y的值赋给a指向地址，此时x的值变为y。

```asm
	movl	12(%ebp), %eax // b -> eax
	movl	-4(%ebp), %edx // temp -> edx
	movl	%edx, (%eax) // edx -> *b
```

将b也就是y的地址赋给eax，temp的值赋给temp。  
最后一句是将temp的值赋给b指向的位置，也就是temp赋给y。

所以上面总结起来就是：

	1. x -> temp
	2. y -> x
	3. temp -> y
	
所以x和y的值被交换了。


综合上面，C语言的地址调用没有任何神秘之处。在这里我们更加确定，C语言没有所谓的传址，一切都是传值。
			