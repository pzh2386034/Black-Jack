---
title: linux锁实现 
date: 2020-02-15
categories:
- mutex
tags:
- mutex,rw_mutex,con_mutex
---

## 前沿

>上一篇大概看了下mutex在glibc的实现，glibc中主要对不同类型的锁做了不同处理；如果锁空闲，则在glibc层就可以完成加锁，并返回客户；但是如果锁被占用，再次lock，则需要陷入内核，使用futex系统库，本篇主要看下futex内核实现

### 上篇主要内容

### 本篇主要内容


## 代码分析


### futex_init

``` c++

static unsigned long __read_mostly futex_hashsize;

static struct futex_hash_bucket *futex_queues;
/*初始化futex hash table的每一个hash bucket头*/
static int __init futex_init(void)
{
	unsigned int futex_shift;
	unsigned long i;

#if CONFIG_BASE_SMALL
	futex_hashsize = 16;
#else
	futex_hashsize = roundup_pow_of_two(256 * num_possible_cpus());
#endif

	futex_queues = alloc_large_system_hash("futex", sizeof(*futex_queues),
					       futex_hashsize, 0,
					       futex_hashsize < 256 ? HASH_SMALL : 0,
					       &futex_shift, NULL,
					       futex_hashsize, futex_hashsize);
	futex_hashsize = 1UL << futex_shift;

	futex_detect_cmpxchg();

	for (i = 0; i < futex_hashsize; i++) {
		atomic_set(&futex_queues[i].waiters, 0);
		plist_head_init(&futex_queues[i].chain);
		spin_lock_init(&futex_queues[i].lock);
	}

	return 0;
}
```

### futex

```c++
/* futex有6个形参，ptread_mutex_lock只管制前4个
 * uaddr: 
 * op:futex系统调用类型；FUTEX_WAIT, FUTEXT_WAKE,FUTEX_FD
    FUTEX_REQUEUE:类似基本的唤醒动作，将val3个等待uaddr的进程(线程)移到uaddr2的等待队列中，然后强制让他们阻塞在uaddr2上面
    FUTEX_CMP_REQUEUE:在futex_requeue基础上多一个判断，只有*uaddr与val2相等时才执行操作
    FUTEXT_WAKE_OP:较复杂
    FUTEXT_LOCK_PI/FUTEXT_UNLOCK_PI/FUTEX_TRYLOCK_PI:带优先级继承的futex锁操作
    FUTEXT_WAIT_BITSET/FUTEXT_WAKE_BITSET:在基本的挂起唤醒操作基础上，额外使用一个bitset参数val3；在使用特定bitset进行wait的进程，只能被使用它的bitset超集的wake调用所唤醒
    FUTEXT_WAIT_REQUEUE_PI/FUTEXT_CMP_REQUEUE_PI：带优先级继承版本的FUTECX_WAIT_REQUEUE
 **
 */
SYSCALL_DEFINE6(futex, u32 __user *, uaddr, int, op, u32, val,
		struct timespec __user *, utime, u32 __user *, uaddr2,
		u32, val3)
{
	struct timespec ts;
	ktime_t t, *tp = NULL;
	u32 val2 = 0;
	int cmd = op & FUTEX_CMD_MASK;

	if (utime && (cmd == FUTEX_WAIT || cmd == FUTEX_LOCK_PI ||
		      cmd == FUTEX_WAIT_BITSET ||
		      cmd == FUTEX_WAIT_REQUEUE_PI)) {
              /*将时间参数从用户态copy到内核态*/
		if (copy_from_user(&ts, utime, sizeof(ts)) != 0)
			return -EFAULT;
            /*校验时间参数*/
		if (!timespec_valid(&ts))
			return -EINVAL;
        /*强制转换时间参数*/
		t = timespec_to_ktime(ts);
		if (cmd == FUTEX_WAIT)
			t = ktime_add_safe(ktime_get(), t);
		tp = &t;
	}
	/*
	 * requeue parameter in 'utime' if cmd == FUTEX_*_REQUEUE_*.
	 * number of waiters to wake in 'utime' if cmd == FUTEX_WAKE_OP.
     * 当FUTEX_*_REQUEUE_*时，utime用来作为与uaddr进行条件判断的参数
     * FUTEX_WAKE_OP，utime表示要唤醒的进程数目
	 */
	if (cmd == FUTEX_REQUEUE || cmd == FUTEX_CMP_REQUEUE ||
	    cmd == FUTEX_CMP_REQUEUE_PI || cmd == FUTEX_WAKE_OP)
		val2 = (u32) (unsigned long) utime;

	return do_futex(uaddr, op, val, tp, uaddr2, val2, val3);
}
```

### do_futex

```c++
long do_futex(u32 __user *uaddr, int op, u32 val, ktime_t *timeout,
		u32 __user *uaddr2, u32 val2, u32 val3)
{
	int cmd = op & FUTEX_CMD_MASK;
	unsigned int flags = 0;
    /*判断是否为共享锁*/
	if (!(op & FUTEX_PRIVATE_FLAG))
		flags |= FLAGS_SHARED;

	if (op & FUTEX_CLOCK_REALTIME) {
		flags |= FLAGS_CLOCKRT;
		if (cmd != FUTEX_WAIT_BITSET && cmd != FUTEX_WAIT_REQUEUE_PI)
			return -ENOSYS;
	}

	switch (cmd) {
    /*为什么所有PI相关锁操作需要futex_cmpxchg_enabled支持*/
	case FUTEX_LOCK_PI:
	case FUTEX_UNLOCK_PI:
	case FUTEX_TRYLOCK_PI:
	case FUTEX_WAIT_REQUEUE_PI:
	case FUTEX_CMP_REQUEUE_PI:
		if (!futex_cmpxchg_enabled)
			return -ENOSYS;
	}
   /*根据op参数不同，执行不同分支*/
	switch (cmd) {
	case FUTEX_WAIT:
		val3 = FUTEX_BITSET_MATCH_ANY;
	case FUTEX_WAIT_BITSET:
		return futex_wait(uaddr, flags, val, timeout, val3);
	case FUTEX_WAKE:
		val3 = FUTEX_BITSET_MATCH_ANY;
	case FUTEX_WAKE_BITSET:
		return futex_wake(uaddr, flags, val, val3);
	case FUTEX_REQUEUE:
		return futex_requeue(uaddr, flags, uaddr2, val, val2, NULL, 0);
	case FUTEX_CMP_REQUEUE:
		return futex_requeue(uaddr, flags, uaddr2, val, val2, &val3, 0);
	case FUTEX_WAKE_OP:
		return futex_wake_op(uaddr, flags, uaddr2, val, val2, val3);
	case FUTEX_LOCK_PI:
		return futex_lock_pi(uaddr, flags, timeout, 0);
	case FUTEX_UNLOCK_PI:
		return futex_unlock_pi(uaddr, flags);
	case FUTEX_TRYLOCK_PI:
		return futex_lock_pi(uaddr, flags, NULL, 1);
	case FUTEX_WAIT_REQUEUE_PI:
		val3 = FUTEX_BITSET_MATCH_ANY;
		return futex_wait_requeue_pi(uaddr, flags, val, timeout, val3,
					     uaddr2);
	case FUTEX_CMP_REQUEUE_PI:
		return futex_requeue(uaddr, flags, uaddr2, val, val2, &val3, 1);
	}
	return -ENOSYS;
}
```

### futex_wait

```c++
static int futex_wait(u32 __user *uaddr, unsigned int flags, u32 val,
		      ktime_t *abs_time, u32 bitset)
{
	struct hrtimer_sleeper timeout, *to = NULL;
	struct restart_block *restart;
	struct futex_hash_bucket *hb;
	struct futex_q q = futex_q_init;
	int ret;

	if (!bitset)
		return -EINVAL;
	q.bitset = bitset;

	if (abs_time) {
		to = &timeout;

		hrtimer_init_on_stack(&to->timer, (flags & FLAGS_CLOCKRT) ?
				      CLOCK_REALTIME : CLOCK_MONOTONIC,
				      HRTIMER_MODE_ABS);
		hrtimer_init_sleeper(to, current);
		hrtimer_set_expires_range_ns(&to->timer, *abs_time,
					     current->timer_slack_ns);
	}

retry:
	/*
	 * Prepare to wait on uaddr. On success, holds hb lock and increments
	 * q.key refs.
	 */
	ret = futex_wait_setup(uaddr, val, flags, &q, &hb);
	if (ret)
		goto out;

	/* queue_me and wait for wakeup, timeout, or a signal. */
	futex_wait_queue_me(hb, &q, to);

	/* If we were woken (and unqueued), we succeeded, whatever. */
	ret = 0;
	/* unqueue_me() drops q.key ref */
	if (!unqueue_me(&q))
		goto out;
	ret = -ETIMEDOUT;
	if (to && !to->task)
		goto out;

	/*
	 * We expect signal_pending(current), but we might be the
	 * victim of a spurious wakeup as well.
	 */
	if (!signal_pending(current))
		goto retry;

	ret = -ERESTARTSYS;
	if (!abs_time)
		goto out;

	restart = &current->restart_block;
	restart->fn = futex_wait_restart;
	restart->futex.uaddr = uaddr;
	restart->futex.val = val;
	restart->futex.time = abs_time->tv64;
	restart->futex.bitset = bitset;
	restart->futex.flags = flags | FLAGS_HAS_TIMEOUT;

	ret = -ERESTART_RESTARTBLOCK;

out:
	if (to) {
		hrtimer_cancel(&to->timer);
		destroy_hrtimer_on_stack(&to->timer);
	}
	return ret;
}
```


## 附录

### futex_hash_bucket

```c++
/*
 * Hash buckets are shared by all the futex_keys that hash to the same
 * location.  Each key may have multiple futex_q structures, one for each task
 * waiting on a futex.
 * 内核通过hash table维护所有挂起阻塞在futex变量上的进程(线程), 不同futex变量会根据其地址标识计算出一个hashkey->hash bucket chain，
 * 所有挂起阻塞在同一个futex变量的所有进程(线程)会定位到同一个hash bucket chain上，plist_head chain即为链表头
 * 一个futex变量对应一个bucket
 */
struct futex_hash_bucket {
	atomic_t waiters;
	spinlock_t lock;/*自旋锁，控制chain的访问*/
    /*优先级链，futex使用优先级链来实现等待队列
     *为实现优先级继承，解决优先级翻转问题？
     */
	struct plist_head chain;
} ____cacheline_aligned_in_smp;

```

### futex_q

```c++
/* 内核通过futex_q结构将一个挂起的进程(线程)和一个futex变量绑定在一起
 * 每个futex变量可以对应好多进程(线程)
 * 每个futex_q对应一个进程，通过plist_node链入它属于的hash bucket中
 */
struct futex_q {
	struct plist_node list;

	struct task_struct *task;
	spinlock_t *lock_ptr;/*hash bucket 锁*/
	union futex_key key;/*futex变量地址标识*/
    /*下面3个变量和优先级继承有关系*/
	struct futex_pi_state *pi_state;
	struct rt_mutex_waiter *rt_waiter;
	union futex_key *requeue_pi_key;
	u32 bitset;
}
```

### futex_key

```c++
union futex_key {
	struct {
		unsigned long pgoff;
		struct inode *inode;
		int offset;
	} shared;/*不同进程间通过文件共享futex变量，表明该变量在文件中的位置*/
	struct {
		unsigned long address;
		struct mm_struct *mm;
		int offset;
	} private;/*同一进程不同线程共享futex变量，表明变量在进程地址空间中的位置*/
	struct {
		unsigned long word;
		void *ptr;
		int offset;
	} both;
};
```
