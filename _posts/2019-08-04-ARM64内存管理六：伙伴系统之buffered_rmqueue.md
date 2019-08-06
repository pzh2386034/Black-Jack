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
 * 从伙伴系统中分配内存，分为两类：
 * 1. order=0: 尝试直接从CPU缓存中分配
 *     如果CPU缓存中没有空闲内存，则首先从伙伴系统中申请bulk个order=0的空闲内存
 * 2. order>0: 尝试从zone的伙伴系统中分配内存__rmqueue()
 *     从free_list[order]开始找空闲内存，找不到则尝试更高一阶，直到找到为止
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
	/* 如果请求order=0，则从cpu cache中获取空闲内存 */
		struct per_cpu_pages *pcp;
		struct list_head *list;

		local_irq_save(flags);
		/* 获取属于此zone的per_cpu_pages指针，结构体见附录 */
		pcp = &this_cpu_ptr(zone->pageset)->pcp;
		/* 获取需要的类型的页框的高速缓存链表，高速缓存中也区分migratetype类型的链表，链表中保存的页框对应的页描述符 */
		list = &pcp->lists[migratetype];
		if (list_empty(list)) {
		/* 如果当前migratetype的每CPU高速缓存链表中没有空闲的页框，从伙伴系统中获取batch个页框加入到这个链表中
		 * batch保存在每CPU高速缓存描述符中，在rmqueue_bulk中是每次要1个页框，要batch次，也就是这些页框是离散的 
		 */
			pcp->count += rmqueue_bulk(zone, 0,
					pcp->batch, list,
					migratetype, cold);
			if (unlikely(list_empty(list)))
				goto failed;
		}
		
		if (cold)
		/* 需要冷的高速缓存，则从每CPU高速缓存的双向链表的后面开始分配 */
			page = list_entry(list->prev, struct page, lru);
		else
		/* 需要热的高速缓存，则从每CPU高速缓存的双向链表的前面开始分配，因为释放时会从链表头插入，所以链表头是热的高速缓存 */
			page = list_entry(list->next, struct page, lru);
		/* 从每CPU高速缓存链表中拿出来 */
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
		/* 从伙伴系统中获取连续页框，返回第一个页的页描述符 */
		page = __rmqueue(zone, order, migratetype);
		spin_unlock(&zone->lock);
		if (!page)
			goto failed;
		 /* 统计，减少zone的free_pages数量统计，因为里面使用加法，所以这里传进负数 */
		__mod_zone_freepage_state(zone, -(1 << order),
					  get_freepage_migratetype(page));
	}
	/* 分配成功后，要统计入NR_ALLOC_BATCH中 */
	__mod_zone_page_state(zone, NR_ALLOC_BATCH, -(1 << order));
	/* 如果经过这次分配，vm_state[NR_ALLOC_BATCH]中已经没有页框了，则标记此zone为ZONE_FAIR_DEPLETED,再上一篇中有介绍使用地方 */
	if (atomic_long_read(&zone->vm_stat[NR_ALLOC_BATCH]) <= 0 &&
	    !test_bit(ZONE_FAIR_DEPLETED, &zone->flags))
		set_bit(ZONE_FAIR_DEPLETED, &zone->flags);

	__count_zone_vm_events(PGALLOC, zone, 1 << order);
	/* 统计 */
	zone_statistics(preferred_zone, zone, gfp_flags);
	local_irq_restore(flags);

	VM_BUG_ON_PAGE(bad_range(zone, page), page);
	/* 返回首页描述符 */
	return page;

failed:
	local_irq_restore(flags);
	return NULL;
}
```

### rmqueue_bulk

``` c++
/*
 * cpu缓存无空闲内存时，从伙伴系统申请bulk个page，通过List_add加入CPU缓存队列
 * __rmqueue可以申请任意阶内存，此处是order=0，重复bulk次，因此是bulk个零散页面
 */
static int rmqueue_bulk(struct zone *zone, unsigned int order,
			unsigned long count, struct list_head *list,
			int migratetype, bool cold)
{
	int i;

	spin_lock(&zone->lock);
	for (i = 0; i < count; ++i) {
		struct page *page = __rmqueue(zone, order, migratetype);
		if (unlikely(page == NULL))
			break;

		if (likely(!cold))
			list_add(&page->lru, list);
		else
			list_add_tail(&page->lru, list);
		list = &page->lru;
		if (is_migrate_cma(get_freepage_migratetype(page)))
			__mod_zone_page_state(zone, NR_FREE_CMA_PAGES,
					      -(1 << order));
	}
	__mod_zone_page_state(zone, NR_FREE_PAGES, -(i << order));
	spin_unlock(&zone->lock);
	return i;
}
```

### __rmqueue

在__rmqueue中，会尝试从free_list中获取空闲内存页，主要有两个步骤:

* 调用__rmqueue_smallest()从free_list[migratetype]中获取空闲页;

* 如果分配失败，则调用__rmqueue_fallback从migratetype的[fallback列表](#fallbacks备选列表)中依次尝试分配

介绍两个struct page的成员变量，page中其余成员变量见参考资料：

* page->_mapcount: 被页表映射的次数，即被多少个进程共享
	
	1. 初始值=-1，如果被一个进程页表映射了，值为0;
	
	2. 如果处于伙伴系统=PAGE_BUDDY_MAPCOUNT_VALUE(-128);
	
	3. 内核通过该值判断page状态;

* page->private: 数据指针

	1. 如果设置了PG_private标志，则private字段指向struct buffer_head;

	2. 如果设置了PG_compound，则指向struct page;

	3. 如果设置了PG_swapcache标志，private存储了该page在交换分区中对应的位置信息swp_entry_t;

	4. 如果_mapcount = PAGE_BUDDY_MAPCOUNT_VALUE，说明该page位于伙伴系统，private存储该伙伴的阶;

	
``` c++
static struct page *__rmqueue(struct zone *zone, unsigned int order,
						int migratetype)
{
	struct page *page;

retry_reserve:
	page = __rmqueue_smallest(zone, order, migratetype);

	if (unlikely(!page) && migratetype != MIGRATE_RESERVE) {
		if (migratetype == MIGRATE_MOVABLE)
			page = __rmqueue_cma_fallback(zone, order);

		if (!page)
			page = __rmqueue_fallback(zone, order, migratetype);

		/*
		 * Use MIGRATE_RESERVE rather than fail an allocation. goto
		 * is used because __rmqueue_smallest is an inline function
		 * and we want just one call site
		 */
		if (!page) {
			migratetype = MIGRATE_RESERVE;
			goto retry_reserve;
		}
	}

	trace_mm_page_alloc_zone_locked(page, order, migratetype);
	return page;
}
```

### __rmqueue_smallest

``` c++
/*
 * Go through the free lists for the given migratetype and remove
 * the smallest available page from the freelists
 */
static inline
struct page *__rmqueue_smallest(struct zone *zone, unsigned int order,
						int migratetype)
{
	unsigned int current_order;
	struct free_area *area;
	struct page *page;

	/* Find a page of the appropriate size in the preferred list */
	for (current_order = order; current_order < MAX_ORDER; ++current_order) {
		area = &(zone->free_area[current_order]);
		/* 如果当前空闲链表为空，则从更高一级的链表中获取空闲页框 */
		if (list_empty(&area->free_list[migratetype]))
			continue;
		/* 获取空闲链表中第一个结点所代表的连续页框 */
		page = list_entry(area->free_list[migratetype].next,
							struct page, lru);
		/* 将页框从空闲链表中删除 */
		list_del(&page->lru);
		/* 
		 * 1. 检查page是否属于伙伴系统,通过page->_mapcount
		 * 2. 将首页框的page->private设置为0; 并初始化page->_mapcount=-1
		 */
		rmv_page_order(page);
		area->nr_free--;
		/* 如果从更高级的页框的链表中分配，这里会将多余的页框放回伙伴系统的链表中
		 * example： 比如我们只需要2个页框，但是这里是从8个连续页框的链表分配给我们的，那其他6个就要拆分为2和4个分别放入链表中
		 */
		expand(zone, page, order, current_order, area, migratetype);
		/* 经过expand返回的page长度已经改变，设置页框的类型与migratetype一致 */
		set_freepage_migratetype(page, migratetype);
		return page;
	}

	return NULL;
}
```

### expand

``` c++
static inline void expand(struct zone *zone, struct page *page,
	int low, int high, struct free_area *area,
	int migratetype)
{
	unsigned long size = 1 << high;
	
	while (high > low) {
	/* 
	 * 从free_list中拿到high阶的空闲内存，需要的low阶的空闲内存
	 * 从high-1阶往下循环知道high = low+1
	 * area为 free_area结构体指针，area--指向frea_area[high-1]
	 */
		area--;
		high--;
		size >>= 1;
		VM_BUG_ON_PAGE(bad_range(zone, &page[size]), &page[size]);

		if (IS_ENABLED(CONFIG_DEBUG_PAGEALLOC) &&
			debug_guardpage_enabled() &&
			high < debug_guardpage_minorder()) {

			set_page_guard(zone, &page[size], high, migratetype);
			continue;
		}
	/* 
	 * page[size]指向多申请的page的首地址，这部分要内存：
	 * 1. 重新放回伙伴系统
	 * 2. 并将该区域free_area计数加1
	 * 3. 设置该page中private=high
	 * 有个疑问：page自己的private变量貌似没有改变？
	 */
		list_add(&page[size].lru, &area->free_list[migratetype]);
		area->nr_free++;
		set_page_order(&page[size], high);
	}
}
```

### __rmqueue_fallback

``` c++
/* 
 * Remove an element from the buddy allocator from the fallback list 
 * __rmqueue_smallest分配失败后，会首先使用__rmqueue_fallback_cma尝试从CMA区分配
 * 如果还分配失败，则使用__rmqueue_fallback依次尝试fallback备选列表中的mirgratetype
 * 从备用mirgratetype中获取到的内存会首先尝试移动到希望的mirgratype，当然可移动是有条件的，主要是为了反碎片
*/
static inline struct page *
__rmqueue_fallback(struct zone *zone, unsigned int order, int start_migratetype)
{
	struct free_area *area;
	unsigned int current_order;
	struct page *page;
	int fallback_mt;
	bool can_steal;

	/* Find the largest possible block of pages in the other list */
	/* 遍历不同order的链表，如果需要分配2个连续页框，则会遍历10,9,8,7,6,5,4,3,2,1这几个链表，注意这里是倒着遍历的 */
	for (current_order = MAX_ORDER-1;
				current_order >= order && current_order <= MAX_ORDER-1;
				--current_order) {
		area = &(zone->free_area[current_order]);
		fallback_mt = find_suitable_fallback(area, current_order,
				start_migratetype, false, &can_steal);
		/* 如果在该阶的frea_area中找不到合适的migratetype，则尝试更低阶的pageblock */
		if (fallback_mt == -1)
			continue;

		page = list_entry(area->free_list[fallback_mt].next,
						struct page, lru);
		if (can_steal)
			steal_suitable_fallback(zone, page, start_migratetype);

		/* Remove the page from the freelists */
		area->nr_free--;
		/* 
		 * 从伙伴系统中拿出来，因为在steal_suitable_fallback已经将新的页框放到了需要的start_mirgatetype的链表中
         * 并且此order并不一定是所需要order的上级，因为order是倒着遍历
         */
		list_del(&page->lru);
		/* 设置page->_mapcount = -1 并且 page->private = 0 */
		rmv_page_order(page);
		/* 如果有多余的页框，则把多余的页框放回伙伴系统中 */
		expand(zone, page, order, current_order, area,
					start_migratetype);
		/* 
		 * 设置获取的页框的类型为新的类型
         * 到这里，page已经是一个2^oder连续页框的内存段，之后就把它返回到申请者就好
         */
		set_freepage_migratetype(page, start_migratetype);

		trace_mm_page_alloc_extfrag(page, order, current_order,
			start_migratetype, fallback_mt);

		return page;
	}

	return NULL;
}
```

### find_suitable_fallback

``` c++
int find_suitable_fallback(struct free_area *area, unsigned int order,
			int migratetype, bool only_stealable, bool *can_steal)
{
	int i;
	int fallback_mt;

	if (area->nr_free == 0)
		return -1;

	*can_steal = false;
	/* 遍历order链表中对应fallbacks优先级的类型链表 */
	for (i = 0;; i++) {
		fallback_mt = fallbacks[migratetype][i];
		/* 这里不能分配MIGRATE_RESERVE类型的内存，这部分内存是保留使用，最后其他的migratetype都没有内存可分配才会分配MIGRATE_RESERVE类型的内存 */
		if (fallback_mt == MIGRATE_RESERVE)
			break;
		/* 链表为空，说明该类型链表页没有内存 */
		if (list_empty(&area->free_list[fallback_mt]))
			continue;
		/* 
		 * 可移动有几种情况：
		 * order>pageblock_order
		 * order>pageblock_order/2 || migratetype == MIGRATE_RECLAIMABLE || migratetype == MIGRATE_UNMOVABLE|| page_group_by_mobility_disabled
		 */
		if (can_steal_fallback(order, migratetype))
			*can_steal = true;

		if (!only_stealable)
			return fallback_mt;

		if (*can_steal)
			return fallback_mt;
	}

	return -1;
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

``` c++
### fallbacks备选列表

static int fallbacks[MIGRATE_TYPES][4] = {
    [MIGRATE_UNMOVABLE]   = { MIGRATE_RECLAIMABLE, MIGRATE_MOVABLE,     MIGRATE_RESERVE },
    [MIGRATE_RECLAIMABLE] = { MIGRATE_UNMOVABLE,   MIGRATE_MOVABLE,     MIGRATE_RESERVE },
#ifdef CONFIG_CMA
    [MIGRATE_MOVABLE]     = { MIGRATE_CMA,         MIGRATE_RECLAIMABLE, MIGRATE_UNMOVABLE, MIGRATE_RESERVE },
    [MIGRATE_CMA]         = { MIGRATE_RESERVE }, /* Never used */
#else
    [MIGRATE_MOVABLE]     = { MIGRATE_RECLAIMABLE, MIGRATE_UNMOVABLE,   MIGRATE_RESERVE },
#endif
    [MIGRATE_RESERVE]     = { MIGRATE_RESERVE }, /* Never used */
#ifdef CONFIG_MEMORY_ISOLATION
    [MIGRATE_ISOLATE]     = { MIGRATE_RESERVE }, /* Never used */
#endif
};
```

## 参考资料

[page结构体成员解析](https://blog.csdn.net/gatieme/article/details/52384636)
