---
title: ARM64内存管理六：伙伴系统之buffered_rmqueue
date: 2019-08-04
categories:
- linux
tags:
- mm,buffered_rmqueue
---


## 前沿

### 往篇回顾

在上一篇中，着重从总体上看，伙伴系统分配内存的大概流程；简单来说就是几句话：

* 首先尝试从prefered_node所在node的zone中直接分配；

* 不行就启动zone_reclaim回收空间，同时扩大到所有zonelist中的zone；

* 再不行就降低水位要求，进行慢速分配；

* 是在还是分配不到就oom；

### cpu高速缓存基本概念

在往下之前，需有先了解一个概念：cpu高速缓存；

* 从硬件上看：高速缓冲存储器Cache是位于CPU与内存之间的临时存储器，它的容量比内存小但交换速度快，一般有L1,L2,L3三级.

* 从linux内核看每个cpu的高速缓存主要涉及几个机构体的管理：struct zone, struct per_cpu_pageset, struct per_cpu_pages, 见[附录](#高速缓存相关结构体).

* struct per_cpu_pages->lists中保存这以migratetype分类的单页框双向链表.

* 如果申请内存时order=0， 则会直接从cpu高速缓存中分配.

* 

### 本篇主要内容

了解完主要内容，接下去就要看下每种不同动作是怎么进行的；

> 本篇主要聚焦于如果找到合适的zone来分配内存，那是如何从伙伴系统中获得内存的呢？

## 代码分析

### buffered_rmqueue

``` c++
/*
 * Allocate a page from the given zone. Use pcplists for order-0 allocations.
 */
static inline
struct page *buffered_rmqueue(struct zone *preferred_zone,
			struct zone *zone, unsigned int order,
			gfp_t gfp_flags, int migratetype)
{
	unsigned long flags;
	struct page *page;
	bool cold = ((gfp_flags & __GFP_COLD) != 0);

	if (likely(order == 0)) {
		struct per_cpu_pages *pcp;
		struct list_head *list;

		local_irq_save(flags);
		pcp = &this_cpu_ptr(zone->pageset)->pcp;
		list = &pcp->lists[migratetype];
		if (list_empty(list)) {
			pcp->count += rmqueue_bulk(zone, 0,
					pcp->batch, list,
					migratetype, cold);
			if (unlikely(list_empty(list)))
				goto failed;
		}

		if (cold)
			page = list_entry(list->prev, struct page, lru);
		else
			page = list_entry(list->next, struct page, lru);

		list_del(&page->lru);
		pcp->count--;
	} else {
		if (unlikely(gfp_flags & __GFP_NOFAIL)) {
			/*
			 * __GFP_NOFAIL is not to be used in new code.
			 *
			 * All __GFP_NOFAIL callers should be fixed so that they
			 * properly detect and handle allocation failures.
			 *
			 * We most definitely don't want callers attempting to
			 * allocate greater than order-1 page units with
			 * __GFP_NOFAIL.
			 */
			WARN_ON_ONCE(order > 1);
		}
		spin_lock_irqsave(&zone->lock, flags);
		page = __rmqueue(zone, order, migratetype);
		spin_unlock(&zone->lock);
		if (!page)
			goto failed;
		__mod_zone_freepage_state(zone, -(1 << order),
					  get_freepage_migratetype(page));
	}

	__mod_zone_page_state(zone, NR_ALLOC_BATCH, -(1 << order));
	if (atomic_long_read(&zone->vm_stat[NR_ALLOC_BATCH]) <= 0 &&
	    !test_bit(ZONE_FAIR_DEPLETED, &zone->flags))
		set_bit(ZONE_FAIR_DEPLETED, &zone->flags);

	__count_zone_vm_events(PGALLOC, zone, 1 << order);
	zone_statistics(preferred_zone, zone, gfp_flags);
	local_irq_restore(flags);

	VM_BUG_ON_PAGE(bad_range(zone, page), page);
	return page;

failed:
	local_irq_restore(flags);
	return NULL;
}
```

## 附录

### 高速缓存相关结构体

``` c++
/* 内存管理区描述符 */
struct zone {
    ........
    /* 实现每CPU页框高速缓存，里面包含每个CPU的单页框的链表 */
    struct per_cpu_pageset __percpu *pageset;
    ........
    /* 对应于伙伴系统中MIGRATE_RESEVE链的页块的数量 */
    int            nr_migrate_reserve_block;
    ........
    /* 标识出管理区中的空闲页框块，用于伙伴系统 */
    /* MAX_ORDER为11，分别代表包含大小为1,2,4,8,16,32,64,128,256,512,1024个连续页框的链表，具体见下面 */
    struct free_area    free_area[MAX_ORDER];
    ......
}

/* 每CPU高速缓存描述符 */
struct per_cpu_pageset {
    /* 核心结构，高速缓存页框结构 */
    struct per_cpu_pages pcp;
#ifdef CONFIG_NUMA
    s8 expire;
#endif
#ifdef CONFIG_SMP
    s8 stat_threshold;
    /* 高速缓存分各种不同的区 */
    s8 vm_stat_diff[NR_VM_ZONE_STAT_ITEMS];
#endif
};

struct per_cpu_pages {
    /* 当前CPU高速缓存中页框个数 */
    int count;        /* number of pages in the list */
    /* 上界，当此CPU高速缓存中页框个数大于high，则会将batch个页框放回伙伴系统 */
    int high;        /* high watermark, emptying needed */
    /* 在高速缓存中将要添加或被删去的页框个数，当链表中页框数量多个上界时会将batch个页框放回伙伴系统，当链表中页框数量为0时则从伙伴系统中获取batch个页框 */
    int batch;        /* chunk size for buddy add/remove */

    /* Lists of pages, one per migrate type stored on the pcp-lists */
    /* 页框的链表，如果需要冷高速缓存，从链表尾开始获取页框，如果需要热高速缓存，从链表头开始获取页框 */
    struct list_head lists[MIGRATE_PCPTYPES];
};
```

