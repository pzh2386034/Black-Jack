---
title: 协程学习：C协程实现之corotine
date: 2020-04-22
categories:coroutine
- 
tags:
- coroutine
---

## 前沿

>最近要优化IPMI消息收发处理能力，看来看去可以实现的比较有效的优化点有2个：报文数据空间申请、释放；使用协程处理收到的报文

## 本篇主要内容

* 学习glibc提供的ucontext组件，主要学习gibc的四个contex函数

* 学习云风的C语言同步协程库

### 进程、线程、协程

* 进程是CPU资源分配的基本单位，对应内核有mm_struct结构体管理进程所有虚拟内存

* 线程是独立运行和独立调度的基本单位，对应内核有task_struct结构体参与内核任务调度

* 协程是用户态轻量级线程，拥有自己的寄存器上下文、栈

### 协程优势及缺点

* 在用户态完成上下文切换，避免陷入内核用户特权级切换级用户态栈到内核态栈的切换

* 切换无需原子操作锁定级同步开销

* 由于系统最小调度单位是线程，协程无法利用多核资源，需要进程配合才能运行在多cpu上；但是可以提高核的单位时间指令执行密度

* OS调度器无法指挥协程间切换，只能由协程主动放弃CPU，控制权回到调度器从而调度另一个协程运行

### 线程上下文

* EIP 寄存器，用来存储 CPU 要读取指令的地址

* ESP 寄存器：指向当前线程栈的栈顶位置

* 其他通用寄存器的内容：包括代表函数参数的 rdi、rsi 等等

* 线程栈中的内存内容

## corotine源码学习

### coroutine_new

``` c++
/**
* 创建一个协程对象
* @param S 该协程所属的调度器
* @param func 该协程函数执行体
* @param ud func的参数
* @return 新建的协程的ID
*/
int coroutine_new(struct schedule *S, coroutine_func func, void *ud) {
	/* 创建一个新的协程结构体,并将status初始化为 READY */
	struct coroutine *co = _co_new(S, func , ud);
	if (S->nco >= S->cap) {
		/* 如果目前协程的数量已经大于调度器的容量，那么进行扩容 */
		int id = S->cap;	/* 新的协程的id直接为当前容量的大小 */
		/* 扩容的方式为，扩大为当前容量的2倍，这种方式和Hashmap的扩容略像 */
		S->co = realloc(S->co, S->cap * 2 * sizeof(struct coroutine *));
		/* 初始化内存 */
		memset(S->co + S->cap , 0 , sizeof(struct coroutine *) * S->cap);
		/* 将协程放入调度器 */
		S->co[S->cap] = co;
		/* 将容量扩大为两倍 */
		S->cap *= 2;
		/* 尚未结束运行的协程的个数 */
		++S->nco; 
		return id;
	} else {
		/* 如果目前协程的数量小于调度器的容量，则取一个为NULL的位置，放入新的协程 */
		int i;
		for (i=0;i<S->cap;i++) {
			/* 
			 * 为什么不 i%S->cap,而是要从nco+i开始呢 
			 * 这其实也算是一种优化策略吧，因为前nco有很大概率都非NULL的，直接跳过去更好
			*/
			int id = (i+S->nco) % S->cap;
			/* 在调度器中找到一个空缺的槽位 */
			if (S->co[id] == NULL) {
				S->co[id] = co;
				++S->nco;
				return id;
			}
		}
	}
	assert(0);
	return -1;
}
```

### coroutine_resume

``` c++
/**
* 切换到对应协程中执行
* 
* @param S 协程调度器
* @param id 协程ID
*/
void coroutine_resume(struct schedule * S, int id) {
	assert(S->running == -1);
	assert(id >=0 && id < S->cap);/* 校验协程ID是否正常 */

    /* 取出要切换的协程 */
	struct coroutine *C = S->co[id];
	if (C == NULL)
		return;

	int status = C->status;
	switch(status) {
	case COROUTINE_READY:
	    /* 调用glibc函数,初始化ucontext_t结构体,保存当前的上下文放到C->ctx里面 */
		getcontext(&C->ctx);
		/* 将当前协程的运行时栈的栈顶设置为S->stack，每个协程都这么设置，这就是所谓的共享栈。（注意，这里是栈顶）*/
		C->ctx.uc_stack.ss_sp = S->stack; 
		C->ctx.uc_stack.ss_size = STACK_SIZE;
		C->ctx.uc_link = &S->main; /* 如果协程执行完，将切换到主协程中执行 */
		S->running = id;
		C->status = COROUTINE_RUNNING;/* 更新当前协程运行状态 */

		/* 设置执行C->ctx函数, 并将S作为参数传进去 */
		uintptr_t ptr = (uintptr_t)S;
		makecontext(&C->ctx, (void (*)(void)) mainfunc, 2, (uint32_t)ptr, (uint32_t)(ptr>>32));

		/* 将当前的上下文放入S->main中，并将C->ctx的上下文替换到当前上下文 */
		swapcontext(&S->main, &C->ctx);
		break;
	case COROUTINE_SUSPEND:
	    /* 将协程所保存的栈的内容，拷贝到当前运行时栈中 */
		/* 其中C->size在yield时有保存 */
		memcpy(S->stack + STACK_SIZE - C->size, C->stack, C->size);
		S->running = id;
		C->status = COROUTINE_RUNNING;
		/* 保存当前上下文到S->main, 并切换到ctx上去，即 mainfunc */
		swapcontext(&S->main, &C->ctx);
		break;
	default:
		assert(0);
	}
}
```

### mainfunc

``` c++
/*
 * 通过low32和hi32 拼出了struct schedule的指针，这里为什么要用这种方式，而不是直接传struct schedule*呢？
 * 因为makecontext的函数指针的参数是int可变列表，在64位下，一个int没法承载一个指针
*/
static void mainfunc(uint32_t low32, uint32_t hi32) {
	uintptr_t ptr = (uintptr_t)low32 | ((uintptr_t)hi32 << 32);
	struct schedule *S = (struct schedule *)ptr;

	int id = S->running;
	struct coroutine *C = S->co[id];
	C->func(S,C->ud);	/* 执行新协程的主函数，中间有可能会有不断的yield */
	_co_delete(C);
	S->co[id] = NULL;
	--S->nco;
	S->running = -1;
}
```

### coroutine_yield

``` c++
/**
* 将当前正在运行的协程让出，切换到主协程上
* @param S 协程调度器
*/
void coroutine_yield(struct schedule * S) {
	/* 取出当前正在运行的协程 */
	int id = S->running;
	assert(id >= 0);

	struct coroutine * C = S->co[id];
	/* 栈是由高往低拓展，正在运行的协程地址一定大于当前栈顶才是有效地址 */
	assert((char *)&C > S->stack);

	/* 将当前运行的协程的栈内容保存起来； 第二个参数为栈底 */
	_save_stack(C,S->stack + STACK_SIZE);
	
	/* 将当前栈的状态改为 COROUTINE_SUSPEND */
	C->status = COROUTINE_SUSPEND;
	S->running = -1;

	/* 所以这里可以看到，只能从协程切换到主协程中 */
	swapcontext(&C->ctx , &S->main);
}
```

### _save_stack

``` c++
/*
* 将本协程的栈内容保存起来
* @top 栈底
*/
static void _save_stack(struct coroutine *C, char *top) {
	/* 这个dummy很关键，是求取整个栈的关键
	 ** 这个非常经典，涉及到linux的内存分布，栈是从高地址向低地址扩展，因此
	 ** S->stack + STACK_SIZE 就是运行时栈的栈底
	 ** dummy，此时在栈中，肯定是位于最底的位置的，即栈顶
	 ** top - &dummy 即整个栈的容量
	 **/
	char dummy = 0;
	/* 确保共享栈使用未超过边界 */
	assert(top - &dummy <= STACK_SIZE);
	/* top - &dummy: 当前协程真实使用的栈空间 */
	if (C->cap < top - &dummy) {
		free(C->stack);				/* 释放原有的保存栈空间 */
		C->cap = top-&dummy;		/* 计算协程真实使用的栈空间 */
		C->stack = malloc(C->cap);	/* 从堆中申请空间，用于保存栈数据 */
	}
	C->size = top - &dummy;
	memcpy(C->stack, &dummy, C->size);/* 将栈中数据copy到申请的堆中保存 */
}
```

### 主协程、子协程执行函数体

``` c++
/* 子协程入口 */
static void
foo(struct schedule * S, void *ud) {
	struct args * arg = ud;
	int start = arg->n;
	int i;
	for (i=0;i<5;i++) {
		printf("coroutine %d : %d\n",coroutine_running(S) , start + i);
		/* 切出当前协程 */
		coroutine_yield(S);
	}
}

/* 主协程函数，每次协程切换都必须回到主协程 */
static void
test(struct schedule *S) {
	struct args arg1 = { 0 };
	struct args arg2 = { 100 };

	/* 创建两个协程 */
	int co1 = coroutine_new(S, foo, &arg1);
	int co2 = coroutine_new(S, foo, &arg2);

	printf("main start\n");
	while (coroutine_status(S,co1) && coroutine_status(S,co2)) {
		/* 使用协程co1 */
		coroutine_resume(S,co1);
		/* 使用协程co2 */
		coroutine_resume(S,co2);
	} 
	printf("main end\n");
}
```

### __getcontext

``` c++

ENTRY(__getcontext)
	/* No need to save r0-r3, d0-d7, or d16-d31.  */
	add	r1, r0, #MCONTEXT_ARM_R4
	stmia   r1, {r4-r11}

	/* Save R13 separately as Thumb can't STM it.  */
	str     r13, [r0, #MCONTEXT_ARM_SP]
	str     r14, [r0, #MCONTEXT_ARM_LR]
	/* Return to LR */
	str     r14, [r0, #MCONTEXT_ARM_PC]
	/* Return zero */
	mov     r2, #0
	str     r2, [r0, #MCONTEXT_ARM_R0]

	/* Save ucontext_t * across the next call.  */
	mov	r4, r0

	/* __sigprocmask(SIG_BLOCK, NULL, &(ucontext->uc_sigmask)) */
	mov     r0, #SIG_BLOCK
	mov     r1, #0
	add     r2, r4, #UCONTEXT_SIGMASK
	bl      PLTJMP(__sigprocmask)

	/* Store FP regs.  Much of the FP code is copied from arm/setjmp.S.  */

#ifdef PIC
	ldr     r2, 1f
	ldr     r1, .Lrtld_global_ro
0:      add     r2, pc, r2
	ldr     r2, [r2, r1]
	ldr     r2, [r2, #RTLD_GLOBAL_RO_DL_HWCAP_OFFSET]
#else
	ldr     r2, .Lhwcap
	ldr     r2, [r2, #0]
#endif

	add	r0, r4, #UCONTEXT_REGSPACE

#ifdef __SOFTFP__
	tst     r2, #HWCAP_ARM_VFP
	beq     .Lno_vfp
#endif

	/* Store the VFP registers.
	   Don't use VFP instructions directly because this code
	   is used in non-VFP multilibs.  */
	/* Following instruction is vstmia r0!, {d8-d15}.  */
	stc     p11, cr8, [r0], #64
	/* Store the floating-point status register.  */
	/* Following instruction is vmrs r1, fpscr.  */
	mrc     p10, 7, r1, cr1, cr0, 0
	str     r1, [r0], #4
.Lno_vfp:

	tst     r2, #HWCAP_ARM_IWMMXT
	beq     .Lno_iwmmxt

	/* Save the call-preserved iWMMXt registers.  */
	/* Following instructions are wstrd wr10, [r0], #8 (etc.)  */
	stcl    p1, cr10, [r0], #8
	stcl    p1, cr11, [r0], #8
	stcl    p1, cr12, [r0], #8
	stcl    p1, cr13, [r0], #8
	stcl    p1, cr14, [r0], #8
	stcl    p1, cr15, [r0], #8
.Lno_iwmmxt:

	/* Restore the clobbered R4 and LR.  */
	ldr	r14, [r4, #MCONTEXT_ARM_LR]
	ldr	r4, [r4, #MCONTEXT_ARM_R4]

	mov	r0, #0

	DO_RET(r14)

END(__getcontext)

#ifdef PIC
1:      .long   _GLOBAL_OFFSET_TABLE_ - 0b - PC_OFS
.Lrtld_global_ro:
	.long   C_SYMBOL_NAME(_rtld_global_ro)(GOT)
#else
.Lhwcap:
	.long   C_SYMBOL_NAME(_dl_hwcap)
#endif


weak_alias(__getcontext, getcontext)
```

### swapcontext

``` c++
ENTRY(swapcontext)

	/* Have getcontext() do most of the work then fix up
	   LR afterwards.  Save R3 to keep the stack aligned.  */
	push	{r0,r1,r3,r14}
	cfi_adjust_cfa_offset (16)
	cfi_rel_offset (r0,0)
	cfi_rel_offset (r1,4)
	cfi_rel_offset (r3,8)
	cfi_rel_offset (r14,12)

	bl	__getcontext
	mov	r4, r0

	pop	{r0,r1,r3,r14}
	cfi_adjust_cfa_offset (-16)
	cfi_restore (r0)
	cfi_restore (r1)
	cfi_restore (r3)
	cfi_restore (r14)

	/* Exit if getcontext() failed.  */
	cmp 	r4, #0
	itt	ne
	movne	r0, r4
	RETINSTR(ne, r14)

	/* Fix up LR and the PC.  */
	str	r13,[r0, #MCONTEXT_ARM_SP]
	str	r14,[r0, #MCONTEXT_ARM_LR]
	str	r14,[r0, #MCONTEXT_ARM_PC]

	/* And swap using swapcontext().  */
	mov	r0, r1
	b	__setcontext

END(swapcontext)
```


## 附录

### ucontext

``` c++
/* 内核ucontex结构体 */
struct ucontext {
	unsigned long	  uc_flags;
	struct ucontext  *uc_link;	/* 指向后继上下文, 运行终止时os会恢复指向的上下文 */
	stack_t		  uc_stack;		/* 当前contex运行的栈信息 */
	struct sigcontext uc_mcontext;/* 保存具体的程序执行上下文，如PC值、堆栈指针、寄存器值等 */
	sigset_t	  uc_sigmask;	/* 本上下文中屏蔽的信号列表，即信号掩码 */
	/* Allow for uc_sigmask growth.  Glibc uses a 1024-bit sigset_t.  */
	int		  __unused[32 - (sizeof (sigset_t) / sizeof (int))];
	/* Last for extensibility.  Eight byte aligned because some
	   coprocessors require eight byte alignment.  */
 	unsigned long	  uc_regspace[128] __attribute__((__aligned__(8)));
};
```

### sigaltstack

``` c++
typedef struct sigaltstack {
	void __user *ss_sp;
	int ss_flags;
	size_t ss_size;
} stack_t;
```

### schedule

``` c++
struct schedule {
	char stack[STACK_SIZE];	/* 运行时栈(共享栈),指向绝对栈顶(相对最低地址) */
	ucontext_t main;		/* 主协程上下文main */
	int nco;				/* 当前激活的协程个数 */
	int cap;				/* 协程管理器的当前最大容量 */
	int running;			/* 正在运行的协程ID */
	struct coroutine **co;	/* 一维数组，用于存放所有可供调度协程指针 */
};
```

### coroutine

``` c++
struct coroutine {
	coroutine_func func;	/* 协程主函数 */
	void *ud;				/* 协程参数 */
	ucontext_t ctx;			/* 协程上下文 */
	struct schedule * sch;	/* 协程所属的调度器 */
	ptrdiff_t cap;			/* 已经分配的内存大小 */
	ptrdiff_t size;			/* 当前协程运行时栈，保存起来后的大小 */
	int status;				/* 当前协程状态 */
	char *stack;			/* 当前协程保存起来的运行时栈,保存在堆中 */
};
```

## 参考资料

[云风coroutine](https://github.com/cloudwu/coroutine)

[libco源码分析](https://www.cyhone.com/articles/analysis-of-libco/)

[云风coroutine源码分析](https://www.cyhone.com/articles/analysis-of-cloudwu-coroutine/)
