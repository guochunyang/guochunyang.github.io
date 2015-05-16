---
layout: post
title: Linux内核如何装载和启动一个可执行程序
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
###exec

本节我们分析exec系统调用的执行过程。  
exec一般和fork调用，常规用法是fork出一个子进程，然后在子进程中执行exec，替换为新的代码。

###do_exec

跟上次的fork类似，这里我们查看do_exec函数。

```c
int do_execve(struct filename *filename,
	const char __user *const __user *__argv,
	const char __user *const __user *__envp)
{
	return do_execve_common(filename, argv, envp);
}


static int do_execve_common(struct filename *filename,
				struct user_arg_ptr argv,
				struct user_arg_ptr envp)
{
	// 检查进程的数量限制

	// 选择最小负载的CPU，以执行新程序
	sched_exec();

	// 填充 linux_binprm结构体
	retval = prepare_binprm(bprm);

	// 拷贝文件名、命令行参数、环境变量
	retval = copy_strings_kernel(1, &bprm->filename, bprm);
	retval = copy_strings(bprm->envc, envp, bprm);
	retval = copy_strings(bprm->argc, argv, bprm);

	// 调用里面的 search_binary_handler 
	retval = exec_binprm(bprm);

	// exec执行成功

}

static int exec_binprm(struct linux_binprm *bprm)
{
	// 扫描formats链表，根据不同的文本格式，选择不同的load函数
	ret = search_binary_handler(bprm);
	// ...
	return ret;
}


```

从上面的代码中可以看到，do_execve调用了do_execve_common，而do_execve_common又主要依靠了exec_binprm，在exec_binprm中又有一个至关重要的函数，叫做search_binary_handler。

所以现在我们的追踪链为：

	do_execve -> do_execve_common -> exec_binprm -> search_binary_handler
	
###search_binary_handler

这个函数的源码如下：

```c
int search_binary_handler(struct linux_binprm *bprm)
{
	// 遍历formats链表
	list_for_each_entry(fmt, &formats, lh) {
		// 应用每种格式的load_binary方法
		retval = fmt->load_binary(bprm);
		// ...
	}
	return retval;
}

```

它的运行逻辑是依次遍历formats中得每种格式，然后根据不同的格式调用响应的load函数。  
例如，对于elf文件执行load_elf_bianry，对于a.out文件执行load_aout_binary函数。  
我们查看下linux_binprm的结构体定义：

```c
struct linux_binfmt {
	struct list_head lh;
	struct module *module;
	int (*load_binary)(struct linux_binprm *);
	int (*load_shlib)(struct file *);
	int (*core_dump)(struct coredump_params *cprm);
	unsigned long min_coredump;	/* minimal dump size */
};
```

我们看到，里面的load_binary本质上是一个函数指针，所以上面的

	retval = fmt->load_binary(bprm);
	
这行代码，实际上对应了不同的函数调用。

因为这里我们追踪的是elf文件，所以接下来我们查看load_elf_bianry函数。

###load_elf_bianry函数

精简后的源码如下：

```c
static int load_elf_binary(struct linux_binprm *bprm)
{
	// ....
	struct pt_regs *regs = current_pt_regs();  // 获取当前进程的寄存器存储位置

	// 获取elf前128个字节
	loc->elf_ex = *((struct elfhdr *)bprm->buf);

	// 检查魔数是否匹配
	if (memcmp(loc->elf_ex.e_ident, ELFMAG, SELFMAG) != 0)
		goto out;

	// 如果既不是可执行文件也不是动态链接程序，就错误退出
	if (loc->elf_ex.e_type != ET_EXEC && loc->elf_ex.e_type != ET_DYN)
		// 
	// 读取所有的头部信息
	// 读入程序的头部分
	retval = kernel_read(bprm->file, loc->elf_ex.e_phoff,
			     (char *)elf_phdata, size);

	// 遍历elf的程序头
	for (i = 0; i < loc->elf_ex.e_phnum; i++) {

		// 如果存在解释器头部
		if (elf_ppnt->p_type == PT_INTERP) {
			// 
			// 读入解释器名
			retval = kernel_read(bprm->file, elf_ppnt->p_offset,
					     elf_interpreter,
					     elf_ppnt->p_filesz);
	
			// 打开解释器文件
			interpreter = open_exec(elf_interpreter);

			// 读入解释器文件的头部
			retval = kernel_read(interpreter, 0, bprm->buf,
					     BINPRM_BUF_SIZE);

			// 获取解释器的头部
			loc->interp_elf_ex = *((struct elfhdr *)bprm->buf);
			break;
		}
		elf_ppnt++;
	}

	// 释放空间、删除信号、关闭带有CLOSE_ON_EXEC标志的文件
	retval = flush_old_exec(bprm);


	setup_new_exec(bprm);

	// 为进程分配用户态堆栈，并塞入参数和环境变量
	retval = setup_arg_pages(bprm, randomize_stack_top(STACK_TOP),
				 executable_stack);
	current->mm->start_stack = bprm->p;

	// 将elf文件映射进内存
	for(i = 0, elf_ppnt = elf_phdata;
	    i < loc->elf_ex.e_phnum; i++, elf_ppnt++) {

		if (unlikely (elf_brk > elf_bss)) {
			unsigned long nbyte;
	            
			// 生成BSS
			retval = set_brk(elf_bss + load_bias,
					 elf_brk + load_bias);
			// ...
		}

		// 可执行程序
		if (loc->elf_ex.e_type == ET_EXEC || load_addr_set) {
			elf_flags |= MAP_FIXED;
		} else if (loc->elf_ex.e_type == ET_DYN) { // 动态链接库
			// ...
		}

		// 创建一个新线性区对可执行文件的数据段进行映射
		error = elf_map(bprm->file, load_bias + vaddr, elf_ppnt,
				elf_prot, elf_flags, 0);

		}

	}

	// 加上偏移量
	loc->elf_ex.e_entry += load_bias;
	// ....


	// 创建一个新的匿名线性区，来映射程序的bss段
	retval = set_brk(elf_bss, elf_brk);

	// 如果是动态链接
	if (elf_interpreter) {
		unsigned long interp_map_addr = 0;

		// 调用一个装入动态链接程序的函数 此时elf_entry指向一个动态链接程序的入口
		elf_entry = load_elf_interp(&loc->interp_elf_ex,
					    interpreter,
					    &interp_map_addr,
					    load_bias);
		// ...
	} else {
		// elf_entry是可执行程序的入口
		elf_entry = loc->elf_ex.e_entry;
		// ....
	}

	// 修改保存在内核堆栈，但属于用户态的eip和esp
	start_thread(regs, elf_entry, bprm->p);
	retval = 0;
	// 
}
```

了解这段程序，得先知道elf文件的大体格式。  
elf文件的开头是它的文件头，我们通过`man elf`可以查看到：

```c
typedef struct {
               unsigned char e_ident[EI_NIDENT];
               uint16_t      e_type;
               uint16_t      e_machine;
               uint32_t      e_version;
               ElfN_Addr     e_entry;
             // ....
           } ElfN_Ehdr;
```

这就是elf文件的头部，它规定了许多与二进制兼容性相关的信息。所以在加载elf文件的时候，必须先加载头部，分析elf的具体信息。 

所以上面程序的大体流程就是：

	1. 分析头部
	2. 查看是否需要动态链接。如果是静态链接的elf文件，那么直接加载文件即可。如果是动态链接的可执行文件，那么需要加载的是动态链接器。
	3. 装载文件，为其准备进程映像。
	4. 为新的代码段设定寄存器以及堆栈信息。

我们可以使用gdb打印下堆栈：

```
#0  start_thread (regs=0xc7869fb4, new_ip=134516010, new_sp=3217801584)
    at arch/x86/kernel/process_32.c:199
#1  0xc1171280 in load_elf_binary (bprm=0xc7bcbd00) at fs/binfmt_elf.c:975
#2  0xc1131bd1 in search_binary_handler (bprm=0xc7bcbd00) at fs/exec.c:1374
#3  0xc113306d in exec_binprm (bprm=<optimized out>) at fs/exec.c:1416
#4  do_execve_common (filename=0xc7bd4000, argv=..., envp=...) at fs/exec.c:1513
#5  0xc113342c in do_execve (__envp=<optimized out>, __argv=<optimized out>, 
    filename=<optimized out>) at fs/exec.c:1555
#6  SYSC_execve (envp=<optimized out>, argv=<optimized out>, filename=<optimized out>)
    at fs/exec.c:1609
#7  SyS_execve (filename=0, argv=0, envp=0) at fs/exec.c:1604
#8  <signal handler called>
#9  0xb7708b5c in ?? ()
```

###start_thread

根据上面打印的堆栈，我们查看start_thread函数：

```c
void
start_thread(struct pt_regs *regs, unsigned long new_ip, unsigned long new_sp)
{
	set_user_gs(regs, 0); // 将用户态的寄存器清空
	regs->fs		= 0;
	regs->ds		= __USER_DS;
	regs->es		= __USER_DS;
	regs->ss		= __USER_DS;
	regs->cs		= __USER_CS;
	regs->ip		= new_ip; // 新进程的运行位置- 动态链接程序的入口处
	regs->sp		= new_sp; // 用户态的栈顶
	regs->flags		= X86_EFLAGS_IF;
	
	set_thread_flag(TIF_NOTIFY_RESUME);
}
```

这里面我们可以看到：

	1. 寄存器清空
	2. 设定寄存器的值，尤其是eip和esp的值。


###新进程的起点

这里跟上次fork的不同点，上次fork一个新进程，子进程的堆栈和父进程完全箱通风，寄存器信息也完全相同，仅仅把系统调用的返回值eax清零。  
而这里，将寄存器清零，堆栈是全新分配的，对于eip，如果是静态链接的可执行文件，那么eip指向该elf文件的文件头e_entry所指的入口地址。  
如果是动态链接，eip指向动态链接器。

###截图
![](http://images.cnitblog.com/blog2015/669654/201504/191242294323742.png)
![](http://images.cnitblog.com/blog2015/669654/201504/191256345104999.png)




###总结

exec本质就是个替换进程代码段的过程，这里面的难点在于elf文件格式的解析，以及新的代码段堆栈信息和寄存器上下文的设定。

###作业署名

郭春阳 原创作品转载请注明出处 ：[《Linux内核分析》MOOC课程](http://mooc.study.163.com/course/USTC-1000029000 )
			