---
title: 协程学习：C语言协程实现
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

## 源码学习

### getcontext

``` c++
```

### makecontext

``` c++
```

### setcontext

``` c++
```

### swapcontext

``` c++
```


## 附录

### ucontext

``` c++
struct ucontext {
	unsigned long	  uc_flags;
	struct ucontext  *uc_link;	/* 指向后继上下文, 运行终止时os会恢复指向的上下文 */
	stack_t		  uc_stack;		/* 本上下文使用的栈 */
	struct sigcontext uc_mcontext;/* 本上下文使用的栈 */
	sigset_t	  uc_sigmask;	/* 本上下文中的阻塞信号集合 */
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

## 参考资料

[云风coroutine](https://github.com/cloudwu/coroutine)
