---
layout: post
title: 分析system_call中断处理过程
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
#分析system_call中断处理过程

上周我们使用gcc内嵌汇编调用系统调用，这次我们具体分析下过程。

###将getpid嵌入menuos

代码从github下载，步骤如下：
	
	1. 增加一个函数，getpid
	2. 在main中添加MenuConfig("getpid","Show Pid", Getpid);
	3. 重新编译 make roofs
	4. 此时启动 执行getpid就可以看到打印出pid为1

见下面两张图：
	![](http://images.cnitblog.com/blog2015/669654/201504/060117242581324.png)
        ![](http://images.cnitblog.com/blog2015/669654/201504/060117547278315.png)




###menuos的原理

其实这个很简单，在上上周我们分析过linux内核的启动过程，1号进程，就是init，它的执行逻辑是/sbin/bin，所以这里的menuos就是编写的init。

这里注意，linux内核源码不包含/sbin/bin，一般的发行版使用的是FreeBSD的版本。

这里的menuos就是自己制作init，然后嵌入linux内核编译出的镜像。
所以在menuos中执行getpid，得到的pid为1，见上图。

###中断处理

中断分为两种，一种是外中断，由外部的IO设备产生，另一种是内中断，也称为异常，由CPU内部产生。

这里涉及到一个很重要的概念，叫做中断上下文。

CPU的运行状态，分为以下三种：

	1. 内核态，运行于进程上下文，内核代表进程运行于内核空间。
	2. 内核态，运行于中断上下文，内核代表硬件运行于内核空间。
	3. 用户态，运行于用户空间。

从上面，我们可以看出，中断上下文不与任何的进程相关联，仅仅运行在内核空间，而且一般只访问内核数据。

这里顺便总结下进程上下文：

	1. 用户级上下文: 正文、数据、用户堆栈以及共享存储区；
	2. 寄存器上下文: 通用寄存器、程序寄存器(IP)、处理器状态寄存器(EFLAGS)、栈指针(ESP)；
	3. 系统级上下文: 进程控制块task_struct、内存管理信息(mm_struct、vm_area_struct、pgd、pte)、内核栈。

其实就是包含三个内容：用户数据、硬件状态（主要是寄存器）、内核数据。  
所以，当发生进程调度时，要将三个上下文全部进行切换。  
当进行系统调用时，仅仅需要切换寄存器上下文。

相比进程上下文，中断上下文仅仅包含一些寄存器的信息。发生中断时，所谓的保护现场和恢复现场，指的就是这些寄存器信息。

###分析system_call

代码如下：

```asm
# system call handler stub
ENTRY(system_call)
	RING0_INT_FRAME			# can't unwind into user space anyway
	ASM_CLAC
	pushl_cfi %eax			# save orig_eax
	SAVE_ALL				# 保存系统寄存器信息
	GET_THREAD_INFO(%ebp)   # 获取thread_info结构的信息
					# system call tracing in operation / emulation
	testl $_TIF_WORK_SYSCALL_ENTRY,TI_flags(%ebp) # 测试是否有系统跟踪
	jnz syscall_trace_entry   # 如果有系统跟踪，先执行，然后再回来
	cmpl $(NR_syscalls), %eax # 比较eax中的系统调用号和最大syscall，超过则无效
	jae syscall_badsys # 无效的系统调用 直接返回
syscall_call:
	call *sys_call_table(,%eax,4) # 调用实际的系统调用程序
syscall_after_call:
	movl %eax,PT_EAX(%esp)		# 将系统调用的返回值eax存储在栈中
syscall_exit:
	LOCKDEP_SYS_EXIT
	DISABLE_INTERRUPTS(CLBR_ANY)	# make sure we don't miss an interrupt
					# setting need_resched or sigpending
					# between sampling and the iret
	TRACE_IRQS_OFF
	movl TI_flags(%ebp), %ecx
	testl $_TIF_ALLWORK_MASK, %ecx	# 检测是否所有工作已完成
	jne syscall_exit_work  			# 未完成，则去执行这些任务

restore_all:
	TRACE_IRQS_IRET			# iret 从系统调用返回

```

这段代码的逻辑主要就是：

	1. 保存寄存器上下文，
	2. 检查系统调用号是否合法
	3. 执行系统调用
	4. 检查是否还有别的工作需要完成
	5. 退出系统调用，返回到用户态

我们继续跟踪里面的syscall_exit_work，它用来处理系统调用之后，未完成的工作

```asm
syscall_exit_work:
	testl $_TIF_WORK_SYSCALL_EXIT, %ecx # 测试syscall的工作完成
	jz work_pending
	TRACE_IRQS_ON
	ENABLE_INTERRUPTS(CLBR_ANY)	# could let syscall_trace_leave() call
					# schedule() instead
	movl %esp, %eax
	call syscall_trace_leave
	jmp resume_userspace
END(syscall_exit_work)
```

这一段的主要作用还是进入work_pending

work_pending代码：

```asm
work_pending:
	testb $_TIF_NEED_RESCHED, %cl  # 判断是否需要调度
	jz work_notifysig   # 不需要则跳转到work_notifysig
work_resched:
	call schedule   # 调度进程
	LOCKDEP_SYS_EXIT
	DISABLE_INTERRUPTS(CLBR_ANY)	# make sure we don't miss an interrupt
					# setting need_resched or sigpending
					# between sampling and the iret
	TRACE_IRQS_OFF
	movl TI_flags(%ebp), %ecx
	andl $_TIF_WORK_MASK, %ecx	# 是否所有工作都已经做完
	jz restore_all  			# 是则退出
	testb $_TIF_NEED_RESCHED, %cl # 测试是否需要调度
	jnz work_resched  			# 重新执行调度代码
```

这段的逻辑很清楚

	1. 先检查是否需要调度，
	2. 如果是，则进行进程调度，之后再次判断。
	3. 如果不需要调度，那么去执行work_notifysig，处理信号

work_notifysig代码：

```asm
work_notifysig:				# 投递信号
#ifdef CONFIG_VM86
	testl $X86_EFLAGS_VM, PT_EFLAGS(%esp) # 判断8086虚模式，也就是保护模式
	movl %esp, %eax
	jne work_notifysig_v86		# 返回到内核空间
1:
#else
	movl %esp, %eax
#endif
	TRACE_IRQS_ON
	ENABLE_INTERRUPTS(CLBR_NONE)
	movb PT_CS(%esp), %bl
	andb $SEGMENT_RPL_MASK, %bl
	cmpb $USER_RPL, %bl
	jb resume_kernel
	xorl %edx, %edx
	call do_notify_resume  # 将信号投递到进程
	jmp resume_userspace  # 恢复用户空间

#ifdef CONFIG_VM86			# 如果是VM86模式，需要保存状态信息
	ALIGN
work_notifysig_v86:
	pushl_cfi %ecx			# save ti_flags for do_notify_resume
	call save_v86_state		# 保存虚模式下的状态
	popl_cfi %ecx
	movl %eax, %esp
	jmp 1b                  # 跳转到上面的代码，执行do_notify_resume
#endif
END(work_pending)
```

这段代码主要是处理信号：

	1. 先检查是否是8086保护模式
	2. 如果是，那么需要先保存虚模式下的状态信息
	3. 然后跳转到之前的代码继续执行
	4. 将信号投递到进程
	5. 恢复用户空间

之后就是返回系统调用

###流程图总结

如图：
![](http://images.cnitblog.com/blog2015/669654/201504/060118330087986.png)



###总结

系统调用中断，本质上也是一个保存状态、进行处理、返回并恢复状态的过程。


###署名信息

	郭春阳 原创作品转载请注明出处 ：《Linux内核分析》MOOC课程 http://mooc.study.163.com/course/USTC-1000029000
			