---
layout: post
title: 分析Linux内核中进程的调度（时间片轮转）-《Linux内核分析》Week2作业
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
###1.环境的搭建：

这个可以参考孟宁老师的github：[mykernel](https://github.com/mengning/mykernel)，这里不再进行赘述。主要是就是下载Linux3.9的代码，然后安装孟宁老师编写的patch，最后进行编译。

###2.代码的解读

课上的代码全部保存在github上，我fork了一份，然后为它加上了详细的注释，参见[mykernel](https://github.com/mengning/mykernel)

###3.代码结构

这里主要有三个文件：
1. mypcb.h 这个头文件定义了进程控制结构PCB
2. mymain.c 这个文件主要是定义了启动N个进程的过程
3. myinterupt.c 这个文件主要是时钟中断函数和进程调度函数的具体实现

###4.进程控制块

这里主要是mypcb.h中定义的结构：

```C
/* CPU-specific state of this task */
// CPU特定状态
struct Thread {
    unsigned long		ip;	// eip寄存器
    unsigned long		sp;	// esp  栈顶寄存器
};

// 进程控制块
typedef struct PCB{
    int pid;	// 进程id
    // 进程状态，未运行、可运行、停止
    volatile long state;	/* -1 unrunnable, 0 runnable, >0 stopped */
    // 进程的栈空间
    char stack[KERNEL_STACK_SIZE];
    /* CPU-specific state of this task */
    struct Thread thread;	// CPU相关的状态
    unsigned long	task_entry;
    struct PCB *next; // 下个进程
}tPCB;
```

我们现在知道，一个进程运行的上下文中，有三个比较重要的寄存器，ebp、esp和eip，这里定义的struct Thread就是用来保存其中的eip和esp，至于ebp，后面可以看到它的存储有另外的方式。

而tPCB这个数据结构的意义，更加明显，他就是用来表示一个进程，里面存储了各种进程相关的信息。

	其中：pid表示进程号
	state表示进程的运行状态（未运行的、可运行的、已经停止的），一共三种
	stack 进程的栈空间，就是上周分析函数调用所使用的空间
	thread，保存eip和esp
	long 进程的代码段
	next 进程的下一个进程
	
这里注意，所有的进程组成了一个链表，而且是`双向链表`。

###进程的启动


```C
void __init my_start_kernel(void)
{
    int pid = 0;
    int i;
    /* Initialize process 0*/
    // 初始化0号进程
    task[pid].pid = pid;
    task[pid].state = 0;/* -1 unrunnable, 0 runnable, >0 stopped */
    // eip指向my_process代码段
    task[pid].task_entry = task[pid].thread.ip = (unsigned long)my_process;
    // esp指向栈底，此时为空栈，栈的地址增长空间为从高到低
    task[pid].thread.sp = (unsigned long)&task[pid].stack[KERNEL_STACK_SIZE-1];
    // 此时链表中的进程只有一个，这是一个循环链表
    task[pid].next = &task[pid];
    /*fork more process */
    // 依次启动NUM-1个进程，总计NUM个进程
    for(i=1;i<MAX_TASK_NUM;i++)
    {
        // 初始化PCB控制块
        memcpy(&task[i],&task[0],sizeof(tPCB));
        task[i].pid = i;
        task[i].state = -1; // 从未运行过的
        task[i].thread.sp = (unsigned long)&task[i].stack[KERNEL_STACK_SIZE-1];
        task[i].next = task[i-1].next; // 将上个进程的next（其实就是0号进程的地址）赋给进程的next
        task[i-1].next = &task[i];  // 将当前进程，链接到上个进程的后面
    }
    /* start process 0 by task[0] */
    pid = 0;
    my_current_task = &task[pid]; // 当前运行的进程为0
    /*
        1. thread.sp -> esp 初始化esp
        2. pushl thread.sp 将sp的值入栈，也就是栈底
        3. pushl thread.ip 将ip，也就是my_process代码段的地址入栈
        4. ret: popl eip 这里执行ret，实质就是popl eip，这一步将上面保存的ip的值，赋给eip
        5. popl ebp 将栈底的地址赋给ebp

        说明几点：
        1. 上面之所以使用ret，是因为eip的值不可以直接修改
        2. 这段代码的目的是运行0号进程，主要是初始化三个定时器，esp、ebp、eip
        3. esp直接初始化
        4. 现将sp和ip的值入栈，后面通过两次出栈，将值赋给eip、ebp
    */
	asm volatile(
    	"movl %1,%%esp\n\t" 	/* set task[pid].thread.sp to esp */
    	"pushl %1\n\t" 	        /* push ebp */
    	"pushl %0\n\t" 	        /* push task[pid].thread.ip */
    	"ret\n\t" 	            /* pop task[pid].thread.ip to eip */
    	"popl %%ebp\n\t"
    	: 
    	: "c" (task[pid].thread.ip),"d" (task[pid].thread.sp)	/* input c or d mean %ecx/%edx*/
	);
}   
```
my_start_kernel可以看做操作系统的入口，在这段代码中主要是这么几件事情：

1.初始化0号进程，其实就是初始化结构体中得各项  
2.利用0号进程的pcb初始化其他进程，其实从这里我们可以看出，每个进程的栈空间是相互独立的。每个进程中的函数调用也是互不干扰的。  
3.利用一段汇编代码，开始`真正运行0号进程`。下面我们重点分析这一段：

```C
asm volatile(
    "movl %1,%%esp\n\t"     /* set task[pid].thread.sp to esp */
    "pushl %1\n\t"          /* push ebp */
    "pushl %0\n\t"          /* push task[pid].thread.ip */
    "ret\n\t"               /* pop task[pid].thread.ip to eip */
    "popl %%ebp\n\t"
    : 
    : "c" (task[pid].thread.ip),"d" (task[pid].thread.sp)   /* input c or d mean %ecx/%edx*/
    );
```

这段代码的工作原理不难：

首先我们必须明确，根据上面的代码，刚创建的进程，ip为my_process代码段的首地址，sp指向栈底元素的位置

	1.将0号进程的sp（其实就是栈底）赋值给esp寄存器
	2.将0号进程的sp压栈
	3.将0号进程的ip压栈
	4.执行ret，出栈，值赋给eip，所以现在的eip寄存器的值为my_process代码段的地址
	5.再次出栈，之前保存的sp赋给ebp寄存器。
	
经过上面几个步骤，CPU中esp、ebp和eip均有了新的值，尤其是eip指向了my_process，所以接下来，开始运行0号进程。

###进程的运行

```C
// 进程的运行逻辑
void my_process(void)
{
    int i = 0;
    while(1)
    {
        i++;
        // 每一千万次循环
        if(i%10000000 == 0)
        {
            // 该进程停止运行
            printk(KERN_NOTICE "this is process %d -\n",my_current_task->pid);
            if(my_need_sched == 1)
            {
                my_need_sched = 0;
        	    my_schedule(); // 执行调度
        	}
            // 该进程开始运行
        	printk(KERN_NOTICE "this is process %d +\n",my_current_task->pid);
        }     
    }
}
```

这段代码是进程的运行逻辑，从这里可以看出，进程运行过程中就在不停的执行i++，每当运行10000000次，进程就检查一次自己是否需要调度（是否需要调度由时钟中断函数决定），如果是，就执行调度函数，切换到下一个进程。


###进程的切换

这段主要分析myinterupt.c中的代码。 

```C
/*
 * Called by timer interrupt.
 * it runs in the name of current running process,
 * so it use kernel stack of current running process
 */
void my_timer_handler(void)
{
    // 时钟中断1000次才有一次调度机会
#if 1
    if(time_count%1000 == 0 && my_need_sched != 1)
    {
        printk(KERN_NOTICE ">>>my_timer_handler here<<<\n");
        my_need_sched = 1; // 将当前进程设置为可以进行调度
    } 
    time_count ++ ;  
#endif
    return;  	
}
```

CPU每个一段时间就产生一个时钟中断，此时就要去调用上面的my_timer_handler函数。上面的注释提示了几点：该函数运行在当前进程的地址空间内，所以它使用当前进程的内核栈空间。  
根据上面的说明，该函数运行在每个进程各自的地址空间内，所以time_count归当前进程所有。所以当time_count达到1000的倍数时，才更改my_need_sched的值，正是这里说明了每个进程运行的时间是1000个CPU时钟。

###进程的切换

对于my_schedule中的代码，我们分两块进行分析：

```C
next = my_current_task->next;
prev = my_current_task;
if(next->state == 0)/* -1 unrunnable, 0 runnable, >0 stopped */
{
	/* switch to next process */
    /*
        %0 prev.sp
        %1 prev.ip
        %2 next.sp
        %3 next.ip

        1. pushl ebp 保存ebp
        2. esp -> prev.sp 保存当前进程的esp到sp
        3. next.sp -> esp 将esp的值改为下一个进程的sp（之前的esp）
        4. $1f -> prev.ip 应该是将eip的值保存到ip
        5. pushl next.ip 新进程的eip放入栈
        6. ret 出栈，将next的ip赋给eip
        7. 切换了进程
        8. popl ebp  恢复ebp （注意这里已经切换了进程）

    */
	asm volatile(	
    	"pushl %%ebp\n\t" 	    /* save ebp */
    	"movl %%esp,%0\n\t" 	/* save esp */
    	"movl %2,%%esp\n\t"     /* restore  esp */
    	"movl $1f,%1\n\t"       /* save eip */	
    	"pushl %3\n\t" 
    	"ret\n\t" 	            /* restore  eip */
    	"1:\t"                  /* next process start here */
    	"popl %%ebp\n\t"
    	: "=m" (prev->thread.sp),"=m" (prev->thread.ip)
    	: "m" (next->thread.sp),"m" (next->thread.ip)
	); 
	my_current_task = next; 
	printk(KERN_NOTICE ">>>switch %d to %d<<<\n",prev->pid,next->pid);   	
}
```

因为进程被初始化时，state均为-1，所以如果state为0，所以该进程之前已经运行过。

我们分析下详细流程：

	1.将ebp寄存器压栈（使用的时prev进程的栈空间）
	2.将esp寄存器的值，保存到prev的sp
	3.将next进程的sp，赋值给esp寄存器。
	4.将eip寄存器的值保存到prev的ip
	5.将next进程的ip压栈
	6.ret，将上面压栈的ip，赋值给eip寄存器
	7.切换进程
	8.从栈顶弹出之前保存的ebp，赋值给ebp，也就是恢复ebp的值。
	
这里注意，`最后一步已经切换了进程，所以这里恢复ebp的值，使用的是上次next进程保存的自己的值！！！`）

我们总结下，上面究竟干了什么？  
1.保存prev进程的ebp、esp和eip  
2.恢复next进程的esp、ebp和eip


下面分析最后一段：

```C
else
{
    next->state = 0;
    my_current_task = next;
    printk(KERN_NOTICE ">>>switch %d to %d<<<\n",prev->pid,next->pid);
    /* switch to new process */
    /*
        %0 prev.sp
        %1 prev.ip
        %2 next.sp
        %3 next.ip

        1. pushl ebp 保存ebp
        2. esp -> prev.sp esp保存到当前进程的sp中
        3. next.sp -> esp 下一个进程的sp赋给esp
        4. next.sp -> ebp 下一个进程的sp赋给ebp
        5. 1 -> prev.ip eip保存到当前进程的ip
        6. pushl next.ip next的ip压栈
        7. ret 出栈，next的ip赋给eip

        跟上面的区别是：本进程初次运行，需要设置ebp，而不是从栈中pop ebp
    */
    asm volatile(   
        "pushl %%ebp\n\t"       /* save ebp */
        "movl %%esp,%0\n\t"     /* save esp */
        "movl %2,%%esp\n\t"     /* restore  esp */
        "movl %2,%%ebp\n\t"     /* restore  ebp */
        "movl $1f,%1\n\t"       /* save eip */  
        "pushl %3\n\t" 
        "ret\n\t"               /* restore  eip */
        : "=m" (prev->thread.sp),"=m" (prev->thread.ip)
        : "m" (next->thread.sp),"m" (next->thread.ip)
    );          
}   
```
	
这里同样是进程切换，但是这里不一样的是，将要运行的是一个新进程。
详细分析如下：

	1.保存当前进程的ebp，压栈
	2.将esp赋给prev的sp
	3.将next进程的sp赋给esp
	4.将next进程的sp赋给ebp
	5.保存eip到prev的ip
	6.将prev的ip进行压栈
	7.ret，出栈，将prev的ip赋给eip
	
上面可以总结为：  
1.保存prev进程的ebp、esp和eip  
2.设置新进程的eip、ebp和esp。  
因为是新进程，所以ebp和esp相同，都是从存储的sp那里取值。

这里和上面的切换有何不同？
主要就是`新进程的ebp不再是从栈顶恢复，而是设置一个新的值`。

###实验截图
![](http://images.cnitblog.com/blog2015/669654/201503/141824440115805.png)



###本周总结

本周的核心是时间片轮转，本周的代码通过时钟中断代码，充分说明了这一点。  
在现代操作系统中的进程调度算法，基本就是基于这一算法所设计的。

通过本周的作业，也更加明确了进程切换的过程，其中最重要的就是进程上下文的切换。

最后，通过本周的学习，我更加熟悉了gcc内联汇编的语法。

###作业署名

郭春阳 原创作品转载请注明出处 ：[《Linux内核分析》MOOC课程](http://mooc.study.163.com/course/USTC-1000029000 )
			