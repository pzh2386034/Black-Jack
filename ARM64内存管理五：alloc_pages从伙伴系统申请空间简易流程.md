---
title: ARM64内存管理五：alloc_pages从伙伴系统申请空间简易流程
date: 2019-08-3 21:18
categories:
- linux
tags:
- mm
---

根据前面几篇介绍，目前为止已经走过了如下几个阶段：

* 在MMU打开前的kernel space、fdt mapping.

* 打开MMU后，通过解析dtb，初始化了memblock, 进入fixmap, memblock时代.

* 之后在经过page_init()->bootmem_init函数初始化zone，zonelist，以及struct page结构体初始化等，buddy sys初具模型.

* 在接下来的mm_init()->mem_init()函数，会把目前空闲的mem通过__free_pages(page, order)添加到buddy sys.

>然后,buddy sys就可以正常申请内存了，接下来看看在伙伴系统的框架内，如何申请、释放内存；以及更进一步页面回收、内存规整、直接回收内存等

buddy system基本概念：

* 伙伴系统是基于struct zone，即每个zone都有自己的伙伴系统，zone->free_area用于描述本zone的伙伴系统状态.

* free_area中对不同的类型的MIGRATE_TYPES有区分管理，具体分类见附录

* free_area维护有最高2^10(MAX_ORDER=10)的连续页框链表.


伙伴系统的页框分配方式主要有两种：

* 快速分配(get_page_from_freelist)：
	
	1. 根据zonelist优先级顺序，以zone的WATER_LOW阀值从相应zone的伙伴系统中分配连续页框.

* 慢速分配(__alloc_pages_slowpath)：(如果不允许__GFP_WAIT，则不能进行慢速分配)
	
	1. 在快速分配失败后，根据zonelist优先级，以zone的WATER_MIN阀值从相应的zone的伙伴系统中分配连续页框.
	
	2. 失败，则唤醒kswapd内核线程进行页框回收、页框的异步压缩、轻同步压缩、OOM等使系统获得更多空闲页框；并在这些操作的过程中不时调用快速分配尝试获取内存.

	
>本篇中主要聚焦快速分配，慢速分配中不同的动作再以后进一步分析





#### shrink_page_list

``` c++
/*
 * shrink_page_list() returns the number of reclaimed pages
 */
static unsigned long shrink_page_list(struct list_head *page_list,
				      struct zone *zone,
				      struct scan_control *sc,
				      enum ttu_flags ttu_flags,
				      unsigned long *ret_nr_dirty,
				      unsigned long *ret_nr_unqueued_dirty,
				      unsigned long *ret_nr_congested,
				      unsigned long *ret_nr_writeback,
				      unsigned long *ret_nr_immediate,
				      bool force_reclaim)
{
    /* 初始化两个链表头 */
	LIST_HEAD(ret_pages);
    /* 这个链表保存本次回收就可以立即进行释放的页 */
	LIST_HEAD(free_pages);
	int pgactivate = 0;
	unsigned long nr_unqueued_dirty = 0;
	unsigned long nr_dirty = 0;
	unsigned long nr_congested = 0;
	unsigned long nr_reclaimed = 0;
	unsigned long nr_writeback = 0;
	unsigned long nr_immediate = 0;
    /* 检查是否需要调度，需要则调度 */
	cond_resched();
    /* 循环page_list，将其中的页一个一个释放 */
	while (!list_empty(page_list)) {
		struct address_space *mapping;
		struct page *page;
		int may_enter_fs;
		enum page_references references = PAGEREF_RECLAIM_CLEAN;
		bool dirty, writeback;
        /* 检查是否需要调度，需要则调度 */
		cond_resched();
        /* 从page_list末尾拿出一个页 */
		page = lru_to_page(page_list);
        /* 将此页从page_list中删除 */
		list_del(&page->lru);
        /* 尝试对此页上锁，如果无法上锁，说明此页正在被其他路径控制，跳转到keep 
         * 对页上锁后，所有访问此页的进程都会加入到zone->wait_table[hash_ptr(page, zone->wait_table_bits)]
         */
		if (!trylock_page(page))
			goto keep;
        /* 在page_list的页一定都是非活动的 */
		VM_BUG_ON_PAGE(PageActive(page), page);
        /* 页所属的zone也要与传入的zone一致 */
		VM_BUG_ON_PAGE(page_zone(page) != zone, page);
        /* 扫描的页数量++ */
		sc->nr_scanned++;
        /* 如果此页被锁在内存中，则跳转到cull_mlocked */   
		if (unlikely(!page_evictable(page)))
			goto cull_mlocked;
        /* 如果扫描控制结构中标识不允许进行unmap操作，并且此页有被映射到页表中，跳转到keep_locked */
		if (!sc->may_unmap && page_mapped(page))
			goto keep_locked;

		/* Double the slab pressure for mapped and swapcache pages */
        /* 对于处于swapcache中或者有进程映射了的页，对sc->nr_scanned再进行一次++
         * swapcache用于在页换出到swap时，页会先跑到swapcache中，当此页完全写入swap分区后，在没有进程对此页进行访问时，swapcache才会释放掉此页
         */
		if (page_mapped(page) || PageSwapCache(page))
			sc->nr_scanned++;
        /* 本次回收是否允许执行IO操作 */
		may_enter_fs = (sc->gfp_mask & __GFP_FS) ||
			(PageSwapCache(page) && (sc->gfp_mask & __GFP_IO));


        /* 检查是否是脏页还有此页是否正在回写到磁盘 
         * 这里面主要判断页描述符的PG_dirty和PG_writeback两个标志
         * 匿名页当加入swapcache后，就会被标记PG_dirty
         * 如果文件所属文件系统有特定is_dirty_writeback操作，则执行文件系统特定的is_dirty_writeback操作
         */
		page_check_dirty_writeback(page, &dirty, &writeback);
        /* 如果是脏页或者正在回写的页，脏页数量++ */
		if (dirty || writeback)
			nr_dirty++;
        /* 是脏页但并没有正在回写，则增加没有进行回写的脏页计数 */
		if (dirty && !writeback)
			nr_unqueued_dirty++;

        /* 获取此页对应的address_space，如果此页是匿名页，则为NULL */
		mapping = page_mapping(page);
        /* 如果此页映射的文件所在的磁盘设备等待队列中有数据(正在进行IO处理)或者此页已经在进行回写回收 */
		if (((dirty || writeback) && mapping &&
		     bdi_write_congested(inode_to_bdi(mapping->host))) ||
		    (writeback && PageReclaim(page)))
            /* 可能比较晚才能进行阻塞回写的页的数量 
              * 因为磁盘设备现在繁忙，队列中有太多需要写入的数据
              */
			nr_congested++;

		/*
		 * If a page at the tail of the LRU is under writeback, there
		 * are three cases to consider.
		 *
		 * 1) If reclaim is encountering an excessive number of pages
		 *    under writeback and this page is both under writeback and
		 *    PageReclaim then it indicates that pages are being queued
		 *    for IO but are being recycled through the LRU before the
		 *    IO can complete. Waiting on the page itself risks an
		 *    indefinite stall if it is impossible to writeback the
		 *    page due to IO error or disconnected storage so instead
		 *    note that the LRU is being scanned too quickly and the
		 *    caller can stall after page list has been processed.
		 *
		 * 2) Global reclaim encounters a page, memcg encounters a
		 *    page that is not marked for immediate reclaim or
		 *    the caller does not have __GFP_FS (or __GFP_IO if it's
		 *    simply going to swap, not to fs). In this case mark
		 *    the page for immediate reclaim and continue scanning.
		 *
		 *    Require may_enter_fs because we would wait on fs, which
		 *    may not have submitted IO yet. And the loop driver might
		 *    enter reclaim, and deadlock if it waits on a page for
		 *    which it is needed to do the write (loop masks off
		 *    __GFP_IO|__GFP_FS for this reason); but more thought
		 *    would probably show more reasons.
		 *
		 * 3) memcg encounters a page that is not already marked
		 *    PageReclaim. memcg does not have any dirty pages
		 *    throttling so we could easily OOM just because too many
		 *    pages are in writeback and there is nothing else to
		 *    reclaim. Wait for the writeback to complete.
		 */
        /* 此页正在进行回写到磁盘，对于正在回写到磁盘的页，是无法进行回收的，除非等待此页回写完成 
         * 此页正在进行回写有两种情况:
         * 1.此页是正常的进行回写(脏太久了)
         * 2.此页是刚不久前进行内存回收时，导致此页进行回写的
         */
		if (PageWriteback(page)) {

			if (current_is_kswapd() &&
			    PageReclaim(page) &&
			    test_bit(ZONE_WRITEBACK, &zone->flags)) {
			/* 情况1：
			 * 当处于kswapd内核进程，且此页正在进行回收，然后zone也表明了多页正在回写
			 * 说明此页是已经回写到磁盘，且正在进行回收的，本次回收不需要对此页进行
			 */
				nr_immediate++;/* 增加nr_immediate计数，此计数说明此页准备就可以回收了 */
				goto keep_locked;

			/* Case 2 above */
			} else if (global_reclaim(sc) ||
			    !PageReclaim(page) || !may_enter_fs) {
            /* 此页正在进行正常的回写(不是因为要回收此页才进行的回写)
             * 两种情况会进入这里:
             * 1.本次是针对整个zone进行内存回收的
             * 2.本次回收不允许进行IO操作
             */
		/* 设置此页正在进行回收，回写完成后会检测PG_reclaim标志，置位则会被放到非活动lru链表末尾 */
				SetPageReclaim(page);
				nr_writeback++;/* 增加需要回写计数器 */

				goto keep_locked;

			/* Case 3 above */
			} else {
		/* 等待此页回写完成，回写完成后，尝试对此页进行回收，只有针对某个memcg进行回收时才会进入这 */
				wait_on_page_writeback(page);
			}
		}

		if (!force_reclaim)
			references = page_check_references(page, sc);
        /*
         * 此次回收时非强制进行回收，那要先判断此页需不需要移动到活动lru链表
         * 如果是匿名页，只要最近此页被进程访问过，则将此页移动到活动lru链表头部，否则回收
         * 如果是映射可执行文件的文件页，只要最近被进程访问过，就放到活动lru链表，否则回收
         * 如果是其他的文件页，如果最近被多个进程访问过，移动到活动lru链表，如果只被1个进程访问过，但是PG_referenced置位了，也放入活动lru链表，其他情况回收
         */
		switch (references) {
		case PAGEREF_ACTIVATE:        /* 将页放到活动lru链表中 */
			goto activate_locked;
		case PAGEREF_KEEP:	/* 页继续保存在非活动lru链表中 */
			goto keep_locked;
        /* 这两个在下面的代码都会尝试回收此页 
         * 注意页所属的vma标记了VM_LOCKED时也会是PAGEREF_RECLAIM，因为后面会要把此页放到lru_unevictable_page链表
         */
		case PAGEREF_RECLAIM:
		case PAGEREF_RECLAIM_CLEAN:
			; /* try to reclaim the page below */
		}

        /* page为匿名页，但是又不处于swapcache中，这里会尝试将其加入到swapcache中 */
		if (PageAnon(page) && !PageSwapCache(page)) {
            /* 如果本次回收禁止io操作，则跳转到keep_locked，让此匿名页继续在非活动lru链表中 */
			if (!(sc->gfp_mask & __GFP_IO))
				goto keep_locked;
            /* 将页page加入到swap_cache，然后这个页被视为文件页，起始就是将页描述符信息保存到以swap页槽偏移量为索引的结点
			 * 设置页描述符的private = swap页槽偏移量生成的页表项swp_entry_t，因为后面会设置所有映射了此页的页表项为此swp_entry_t
			 * 设置页的PG_swapcache标志，表明此页在swapcache中，正在被换出
			 * 标记页page为脏页(PG_dirty)，后面就会被换出
			 */
            /* 执行成功后，页属于swapcache，并且此页的page->_count会++，目前引用此页的进程页表没有设置，进程还是可以正常访问这个页 */
			if (!add_to_swap(page, page_list))
				goto activate_locked;/* 失败，将此页加入到活动lru链表中 */
			may_enter_fs = 1;/* 设置可能会用到文件系统相关的操作 */

            /* 获取此匿名页所在的swapcache的address_space，这个是根据page->private中保存的swp_entry_t获得 */
			mapping = page_mapping(page);
		}

        /* 这里是要对所有映射了此page的页表进行设置
         * 匿名页会把对应的页表项设置为之前获取的swp_entry_t
         */
		if (page_mapped(page) && mapping) {
            /* 通过RMAP对所有映射了此页的进程的页表进行此页的unmap操作
             * ttu_flags基本都有TTU_UNMAP标志
             * 如果是匿名页，那么page->private中是一个带有swap页槽偏移量的swp_entry_t，此后这个swp_entry_t可以转为页表项
             * 执行完此后，匿名页在swapcache中，而对于引用了此页的进程而言，此页已经在swap中
             * 但是当此匿名页还没有完全写到swap中时，如果此时有进程访问此页，会将此页映射到此进程页表中，并取消此页放入swap中的操作，放回匿名页的lru链表(在缺页中断中完成)
             * 而对于文件页，只需要清空映射了此页的进程页表的页表项，不需要设置新的页表项
             * 每一个进程unmap此页，此页的page->_count--
             * 如果反向映射过程中page->_count == 0，则释放此页
             */
			switch (try_to_unmap(page, ttu_flags)) {
			case SWAP_FAIL:
				goto activate_locked;
			case SWAP_AGAIN:
				goto keep_locked;
			case SWAP_MLOCK:
				goto cull_mlocked;
			case SWAP_SUCCESS:
				; /* try to free the page below */
			}
		}
        /* 如果页为脏页，有两种页
         * 一种是当匿名页加入到swapcache中时，就被标记为了脏页
         * 一种是脏的文件页
         */
		if (PageDirty(page)) {
            /* 只有kswapd内核线程能够进行文件页的回写操作(kswapd中不会造成栈溢出?)，但是只有当zone中有很多脏页时，kswapd也才能进行脏文件页的回写
             * 此标记说明zone的脏页很多，在回收时隔离出来的页都是没有进行回写的脏页时设置
             * 也就是此zone脏页不够多，kswapd不用进行回写操作
             * 当短时间内多次对此zone执行内存回收后，这个ZONE_DIRTY就会被设置，这样做的理由是: 优先回收匿名页和干净的文件页，说不定回收完这些zone中空闲内存就足够了，不需要再进行内存回收了
             * 而对于匿名页，无论是否是kswapd都可以进行回写
             */
			if (page_is_file_cache(page) &&
					(!current_is_kswapd() ||
					 !test_bit(ZONE_DIRTY, &zone->flags))) {
                /* 增加优先回收页的数量 */
				inc_zone_page_state(page, NR_VMSCAN_IMMEDIATE);
                /* 设置此页需要回收，这样当此页回写完成后，就会被放入到非活动lru链表尾部 
                 * 不过可惜这里只能等kswapd内核线程对此页进行回写，要么就等系统到期后自动将此页进行回写，非kswapd线程都不能对文件页进行回写
                 */
				SetPageReclaim(page);
                /* 让页移动到非活动lru链表头部，如上所说，当回写完成后，页会被移动到非活动lru链表尾部，而内存回收是从非活动lru链表尾部拿页出来回收的 */
				goto keep_locked;
			}
            /* 当zone没有标记ZONE_DIRTY时，kswapd内核线程则会执行到这里 */
            /* 当page_check_references()获取页的状态是PAGEREF_RECLAIM_CLEAN，则跳到keep_locked
             * 页最近没被进程访问过，但此页的PG_referenced被置位
             */
			if (references == PAGEREF_RECLAIM_CLEAN)
				goto keep_locked;
            /* 回收不允许执行文件系统相关操作，则让页移动到非活动lru链表头部 */
			if (!may_enter_fs)
				goto keep_locked;
            /* 回收不允许进行回写，则让页移动到非活动lru链表头部 */
			if (!sc->may_writepage)
				goto keep_locked;

            /* 将页进行回写到磁盘，这里只是将页加入到块层，调用结束并不是代表此页已经回写完成
             * 主要调用page->mapping->a_ops->writepage进行回写，对于匿名页，也是swapcache的address_space->a_ops->writepage
             * 页被加入到块层回写队列后，会置位页的PG_writeback，回写完成后清除PG_writeback位，所以在同步模式回写下，结束后PG_writeback位是0的，而异步模式下，PG_writeback很可能为1
             * 此函数中会清除页的PG_dirty标志
             * 会标记页的PG_reclaim
             * 成功将页加入到块层后，页的PG_lock位会清空
             * 也就是在一个页成功进入到回收导致的回写过程中，它的PG_writeback和PG_reclaim标志会置位，而它的PG_dirty和PG_lock标志会清除
             * 而此页成功回写后，它的PG_writeback和PG_reclaim位都会被清除
             */
			switch (pageout(page, mapping, sc)) {
			case PAGE_KEEP:
				goto keep_locked;                /* 页会被移动到非活动lru链表头部 */
			case PAGE_ACTIVATE:
				goto activate_locked;/* 页会被移动到活动lru链表 */
			case PAGE_SUCCESS:
                /* 到这里，页的锁已经被释放，也就是PG_lock被清空 
                 * 对于同步回写(一些特殊文件系统只支持同步回写)，这里的PG_writeback、PG_reclaim、PG_dirty、PG_lock标志都是清0的
                 * 对于异步回写，PG_dirty、PG_lock标志都是为0，PG_writeback、PG_reclaim可能为1可能为0(回写完成为0，否则为1)
                 */
                /* 如果PG_writeback被置位，说明此页正在进行回写，这种情况是异步才会发生 */
				if (PageWriteback(page))
					goto keep;
                /* 此页为脏页，这种情况发生在此页最近又被写入了，让其保持在非活动lru链表中 
                 * 还有一种情况，就是匿名页加入到swapcache前，已经没有进程映射此匿名页了，而加入swapcache时不会判断
                 * 但是当对此匿名页进行回写时，会判断此页加入swapcache前是否有进程映射了，如果没有，此页可以直接释放，不需要写入磁盘
                 * 所以在此匿名页回写过程中，就会将此页从swap分区的address_space中的基树拿出来，然后标记为脏页，到这里就会进行判断脏页，之后会释放掉此页
                 */
				if (PageDirty(page))
					goto keep;

                /* 尝试上锁，因为在pageout中会释放page的锁，主要是PG_lock标志 */
				if (!trylock_page(page))
					goto keep;
				if (PageDirty(page) || PageWriteback(page))
					goto keep_locked;
				mapping = page_mapping(page);                /* 获取page->mapping */
            /* 这个页不是脏页，不需要回写，这种情况只发生在文件页，匿名页当加入到swapcache中时就被设置为脏页 */
			case PAGE_CLEAN:
				; /* try to free the page below */
			}
		}

        /* 这里的情况只有页已经完成回写后才会到达这里，比如同步回写时(pageout在页回写完成后才返回)，异步回写时，在运行到此之前已经把页回写到磁盘
         * 没有完成回写的页不会到这里，在pageout()后就跳到keep了
         */
        /* 通过页描述符的PAGE_FLAGS_PRIVATE标记判断是否有buffer_head，这个只有文件页有
         * 这里不会通过page->private判断，原因是，当匿名页加入到swapcache时，也会使用page->private，而不会标记PAGE_FLAGS_PRIVATE
         * 只有文件页会使用这个PAGE_FLAGS_PRIVATE，这个标记说明此文件页的page->private指向struct buffer_head链表头
         */
		if (page_has_private(page)) {
            /* 因为页已经回写完成或者是干净不需要回写的页，释放page->private指向struct buffer_head链表，释放后page->private = NULL 
             * 释放时必须要保证此页的PG_writeback位为0，也就是此页已经回写到磁盘中了
             */
			if (!try_to_release_page(page, sc->gfp_mask))
				goto activate_locked;                /* 释放失败，把此页移动到活动lru链表 */
            /* 一些特殊的页的mapping为空，比如一些日志的缓冲区，对于这些页如果引用计数为1则进行处理 */
			if (!mapping && page_count(page) == 1) {
				unlock_page(page);
				if (put_page_testzero(page))/* 对page->_count--，并判断是否为0，如果为0则释放掉此页 */
					goto free_it;
				else {
					/*
					 * rare race with speculative reference.
					 * the speculative reference will free
					 * this page shortly, so we may
					 * increment nr_reclaimed here (and
					 * leave it off the LRU).
					 */
					nr_reclaimed++;
					continue;
				}
			}
		}
        /* 
         * 经过上面的步骤，在没有进程再对此页进行访问的前提下，page->_count应该为2
         * 表示只有将此页隔离出lru的链表和加入address_space的基树中对此页进行了引用，已经没有其他地方对此页进行引用，
         * 然后将此页从address_space的基数中移除，然后page->_count - 2，这个页现在就只剩等待着被释放掉了
         * 如果是匿名页，则是对应的swapcache的address_space的基树
         * 如果是文件页，则是对应文件的address_space的基树
         * 当page->_count为2时，才会将此页从address_space的基数中移除，然后再page->_count - 2
         * 相反，如果此page->_count不为2，说明unmap后又有进程访问了此页，就不对此页进行释放了
         * 同时，这里对于脏页也不能够进行释放，想象一下，如果一个进程访问了此页，写了数据，又unmap此页，那么此页的page->_count为2，同样也可以释放掉，但是写入的数据就丢失了
         * 成功返回1，失败返回0
         */
		if (!mapping || !__remove_mapping(mapping, page, true))
			goto keep_locked;

		/*
		 * At this point, we have no other references and there is
		 * no way to pick any more up (removed from LRU, removed
		 * from pagecache). Can use non-atomic bitops now (and
		 * we obviously don't have to worry about waking up a process
		 * waiting on the page lock, because there are no references.
		 */
		__clear_page_locked(page);
free_it:
        /* page->_count为0才会到这 */
        
        /* 此页可以马上回收，会把它加入到free_pages链表
         * 到这里的页有三种情况，本次进行同步回写的页，干净的不需要回写的页，之前异步回收时完成异步回写的页
         * 之前回收进行异步回写的页，不会立即释放，因为上次回收时，对这些页进行的工作有: 
         * 匿名页: 加入swapcache，反向映射修改了映射了此页的进程页表项，将此匿名页回写到磁盘，将此页保存到非活动匿名页lru链表尾部
         * 文件页: 反向映射修改了映射了此页的进程页表项，将此文件页回写到磁盘，将此页保存到非活动文件页lru链表尾部
         * 也就是异步情况这两种页都没有进行实际的回收，而在这些页回写完成后，再进行回收时，这两种页的流程都会到这里进行回收
         * 也就是本次能够真正回收到的页，可能是之前进行回收时已经处理得差不多并回写完成的页
         */
		nr_reclaimed++;

		/*
		 * Is there need to periodically free_page_list? It would
		 * appear not as the counts should be low
		 */
		list_add(&page->lru, &free_pages);/* 加入到free_pages链表 */
		continue;

cull_mlocked:
        /* 当前页被mlock所在内存中的情况 */

        /* 此页为匿名页并且已经放入了swapcache中了 */
		if (PageSwapCache(page))
            /* 从swapcache中释放本页在基树的结点，会page->_count-- */
			try_to_free_swap(page);
		unlock_page(page);
        /* 把此页放回到lru链表中，因为此页已经被隔离出来了
         * 加入可回收lru链表后page->_count++，但同时也会释放隔离的page->_count--
         * 加入unevictablelru不会进行page->_count++
         */
		list_add(&page->lru, &ret_pages);
		continue;

activate_locked:
		/* Not a candidate for swapping, so reclaim swap space. */
        /* 这种是持有页锁(PG_lock)，并且需要把页移动到活动lru链表中的情况 */

        /* 如果此页为匿名页并且放入了swapcache中，并且swap使用率已经超过了50% */
		if (PageSwapCache(page) && vm_swap_full())
			try_to_free_swap(page);/* 将此页从swapcache的基树中拿出来 */
		VM_BUG_ON_PAGE(PageActive(page), page);
		SetPageActive(page);/* 设置此页为活动页 */;
		pgactivate++;/* 需要放回到活动lru链表的页数量 */
keep_locked:
        /* 希望页保持在原来的lru链表中，并且持有页锁的情况 */

        /* 释放页锁(PG_lock) */
		unlock_page(page);
keep:
        /* 希望页保持在原来的lru链表中的情况 */

        /* 把页加入到ret_pages链表中 */
		list_add(&page->lru, &ret_pages);
		VM_BUG_ON_PAGE(PageLRU(page) || PageUnevictable(page), page);
	}

	mem_cgroup_uncharge_list(&free_pages);
    /* 将free_pages中的页释放 */
	free_hot_cold_page_list(&free_pages, true);
    /* 将ret_pages链表加入到page_list中 */
	list_splice(&ret_pages, page_list);
	count_vm_events(PGACTIVATE, pgactivate);

	*ret_nr_dirty += nr_dirty;
	*ret_nr_congested += nr_congested;
	*ret_nr_unqueued_dirty += nr_unqueued_dirty;
	*ret_nr_writeback += nr_writeback;
	*ret_nr_immediate += nr_immediate;
	return nr_reclaimed;
}
```


## 参考资料

[tolimit-内存回收(整体流程)](https://www.cnblogs.com/tolimit/p/5435068.html)
