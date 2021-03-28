#Challenge 1 用户态和内核态切换
通过编写c++内联汇编函数，来实现用户态和内核态的切换。

已知初始化内核的函数如下：

	    static void
			switch_test(void) {
			print_cur_status(); // print 当前 cs/ss/ds 等寄存器状态
			cprintf("+++ switch to user mode +++\n");
			switch_to_user(); // switch to user mode
			print_cur_status();
			cprintf("+++ switch to kernel mode +++\n");
			switch_to_kernel(); // switch to kernel mode
			print_cur_status();
		}
而在init.c里面没有switch\_to\_user函数的内容以及switch\_to\_kernel函数的内容，因此这些部分需要编程。

##1.调到用户态
通过编写switch\_to\_user的内容来使在内核初始化的时候可以切换到用户态。

	asm volatile (
	    "sub $0x8, %%esp \n"
	    "int %0 \n"
	    "movl %%ebp, %%esp"
	    : 
	    : "i"(T_SWITCH_TOU)
	);
通过编写内联汇编语句来实现切换，用中断处理int 0来实现，需要输入的“i”也就是T\_SWITCH_TOU。调整栈顶指针和栈底指针开辟程序空间之后，进行中断来实现切换。
在trap.c中要修改t\_switch\_tou的内容
	
	case T_SWITCH_TOU:
        if (tf->tf_cs != USER_CS) {
            switchk2u = *tf;
            switchk2u.tf_cs = USER_CS;
            switchk2u.tf_ds = switchk2u.tf_es = switchk2u.tf_ss = USER_DS;

这段代码可以使原来的状态不是用户态时候，将原来的状态修改成用户态。

	switchk2u.tf_eflags |= FL_IOPL_MASK;
    *((uint32_t*)tf - 1) = (uint32_t)&switchk2u;
将这里的标记确认成为用户态，并且中断返回到正确的栈。

##3.用户态到内核态
用类似的内联汇编处理。

	asm volatile (
	    "int %0 \n"
	    "movl %%ebp, %%esp \n"
	    : 
	    : "i"(T_SWITCH_TOK)
	);
因为切换到的是内核态，不需要更多空间，所以相比内核到用户的内联汇编不再需要在栈指针减去一个值来开辟空间。

然后我们同样需要在trap.c中修改T\_SWITCH_TOK的内容

	case T_SWITCH_TOK:
        if (tf->tf_cs != KERNEL_CS) {
            tf->tf_cs = KERNEL_CS;
            tf->tf_ds = tf->tf_es = KERNEL_DS;
            tf->tf_eflags &= ~FL_IOPL_MASK;
            switchu2k = (struct trapframe *)(tf->tf_esp (sizeof(struct trapframe) - 8));
            memmove(switchu2k, tf, sizeof(struct trapframe) - 8);
            *((uint32_t *)tf - 1) = (uint32_t)switchu2k;
        }
跟之前切换的修改类似。