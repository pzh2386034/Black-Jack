---
title: ARM64内存管理十五：do_page_fault缺页中断
date: 2019-09-15
categories:
- linux-memory
tags:
- do_page_fault,malloc
---

## 前沿

### 往篇回顾

>在上一篇中，主要介绍了系统调用brk的相关处理流程；brk根据传入参数brk和mm->brk的比较可以分为两类操作

* 收缩堆内存空间(do_munmap)

	1. 通过find_vma找到start, end两个地址所在的线性地址区间,判断要释放的区间是否与已存在的vma area有重叠;
	
	2. 有重叠则需要通过__split_vma分割vma area，保留不涉及的地址；分割完成后：
	
	3. 通过detach_vmas_to_be_unmapped()将要释放的vma area从红黑树，内存双向链表中删除，vmacache_invalidate使vmacache无效
	
	4. 通过unmap_region()释放vma area的页框
	
	5. 通过remove_vma_list()刷新管理结构体统计数据

* 扩大堆内存空间(do_brk)

	1. 通过get_unmapped_area()在当前进程的地址空间中查找一个符合len大小的线性区间，且该线性区间必须在addr地址之后
	
	2. 通过find_vma_links遍历每个vma，确定上一步找到的新区间之前的线性区对象的位置；如果addr位于某个现存vma，则调用do_munmap()删除；删除成功后继续查找；
	
	3. 对找到的空闲线性区间通过vma_merge尝试与邻近的线性区间进行合并，成功则更新原有vma_area_struct数据并返回即可；
	
	4. 通过kmem_cache_zalloc()在特定的slab高速缓存vm_area_cachep中为这个线性区分配vm_area_struct结构的描述符，并初始化结构体，更新红黑树，双向链表；
	
	5. 如果当前vma设置了VM_LOCKED字段，那么通过mlock_vma_pages_range()立即为这个线性区分配物理页框，否则，do_brk()结束；
	
	
### linux内存分配函数异同点

用户态/内核态|API|物理地址连续|大小限制|单位|场景
--|:--:|--:|:--:|--:|:--:
用户态|malloc/calloc/realloc/free|不保证|堆内存|byte|calloc初始化为0；realloc改变内存大小
用户态|alloca|不保证|栈内存|byte|向栈申请空间
用户态|mmap/munmap|||byte|将文件利用虚拟内存技术映射到内存中去
用户态|brk,sbrk|不保证|堆内存|byte|虚拟内存到内存的映射
内核态|vmalloc/vfree|虚拟地址连续，物理不确定|vmalloc区域vmalloc_start~vmalloc_end，比kmalloc慢|byte|可能睡眠，不能从中断上下文中调用，或其它不允许阻塞情况下调用
内核态|kmalloc/kcalloc/krealloc/kfree|物理连续|64B-4MB|2^order, normal区|最大/小值由KMALLOC_MIN_SIZE/KMALLOC_SHIFT_MAX，对应64B/4MB，从/proc/slabinfo中的kmalloc-xxxx中分配，建立在kmem_cache_create基础之上
内核态|kmem_cache_create|物理连续|64B-4MB|字节大小需对齐,normal区|便于固定大小数据的频繁分配和释放，分配时从缓存池中获取地址，释放时也不一定真正释放内存。通过slab进行管理
内核态|__get_free_page/__get_free_pages|物理连续|0~1024页|normal区|__get_free_pages基于alloc_pages，但是限定不能使用HIGHMEM
内核态|alloc_page/alloc_pages/free_pages|物理连续|4MB|normal区|CONFIG_FORCE_MAX_ZONEORDER定义了最大页面数2^11，一次能分配到的最大页面数是1024
	
	
### 本篇主要内容

>本文主要分析缺页中断流程，关键函数do_page_fault()
	
	
## 代码分析


### do_page_fault 

``` c++
/*
 * 入参：regs: 指向保存在堆栈中的寄存器；error_code: 异常的错误码
 */
static int __kprobes do_page_fault(unsigned long addr, unsigned int esr,
				   struct pt_regs *regs)
{
	struct task_struct *tsk;
	struct mm_struct *mm;
	int fault, sig, code;
	unsigned long vm_flags = VM_READ | VM_WRITE | VM_EXEC;
	unsigned int mm_flags = FAULT_FLAG_ALLOW_RETRY | FAULT_FLAG_KILLABLE;

	tsk = current;
	mm  = tsk->mm;

	/* Enable interrupts if they were enabled in the parent context. */
	if (interrupts_enabled(regs))
		local_irq_enable();

	/*
	 * in_atomic判断当前状态是否处于中断上下文或者禁止抢占，如果是跳转到no_context；
	 * 如果当前进程没有mm，说明是一个内核线程，跳转到no_context；
	 */
	if (in_atomic() || !mm)
		goto no_context;
	/* cpsr为当前程序状态寄存器，如果M[0:4]均为0，则处于用户模式，详细见附录 */
	if (user_mode(regs))
		mm_flags |= FAULT_FLAG_USER;

	if (esr & ESR_LNX_EXEC) {
		vm_flags = VM_EXEC;
	} else if ((esr & ESR_ELx_WNR) && !(esr & ESR_ELx_CM)) {
		vm_flags = VM_WRITE;
		mm_flags |= FAULT_FLAG_WRITE;
	}

	/*
	 * As per x86, we may deadlock here. However, since the kernel only
	 * validly references user space from well defined areas of the code,
	 * we can bug out early if this is from code which shouldn't.
	 */
	if (!down_read_trylock(&mm->mmap_sem)) {
	/* 获取锁失败 */
	/* 如果在内核态且在exception table查询不到该地址，则跳转到no_context */
		if (!user_mode(regs) && !search_exception_tables(regs->pc))
			goto no_context;
retry:
		down_read(&mm->mmap_sem);/* 用户态则睡眠等待锁持有者释放锁 */
	} else {
		/*
		 * The above down_read_trylock() might have succeeded in which
		 * case, we'll have missed the might_sleep() from down_read().
		 */
		might_sleep();
#ifdef CONFIG_DEBUG_VM
		if (!user_mode(regs) && !search_exception_tables(regs->pc))
			goto no_context;
#endif
	}

	fault = __do_page_fault(mm, addr, mm_flags, vm_flags, tsk);

	/*
	 * If we need to retry but a fatal signal is pending, handle the
	 * signal first. We do not need to release the mmap_sem because it
	 * would already be released in __lock_page_or_retry in mm/filemap.c.
	 */
	if ((fault & VM_FAULT_RETRY) && fatal_signal_pending(current))
		return 0;

	/*
	 * Major/minor page fault accounting is only done on the initial
	 * attempt. If we go through a retry, it is extremely likely that the
	 * page will be found in page cache at that point.
	 */

	perf_sw_event(PERF_COUNT_SW_PAGE_FAULTS, 1, regs, addr);
	if (mm_flags & FAULT_FLAG_ALLOW_RETRY) {
		if (fault & VM_FAULT_MAJOR) {
			tsk->maj_flt++;
			perf_sw_event(PERF_COUNT_SW_PAGE_FAULTS_MAJ, 1, regs,
				      addr);
		} else {
			tsk->min_flt++;
			perf_sw_event(PERF_COUNT_SW_PAGE_FAULTS_MIN, 1, regs,
				      addr);
		}
		if (fault & VM_FAULT_RETRY) {
			/*
			 * Clear FAULT_FLAG_ALLOW_RETRY to avoid any risk of
			 * starvation.
			 */
			mm_flags &= ~FAULT_FLAG_ALLOW_RETRY;
			goto retry;
		}
	}

	up_read(&mm->mmap_sem);

	/*
	 * Handle the "normal" case first - VM_FAULT_MAJOR / VM_FAULT_MINOR
	 */
	if (likely(!(fault & (VM_FAULT_ERROR | VM_FAULT_BADMAP |
			      VM_FAULT_BADACCESS))))
		return 0;

	/*
	 * If we are in kernel mode at this point, we have no context to
	 * handle this fault with.
	 */
	if (!user_mode(regs))
		goto no_context;

	if (fault & VM_FAULT_OOM) {
		/*
		 * We ran out of memory, call the OOM killer, and return to
		 * userspace (which will retry the fault, or kill us if we got
		 * oom-killed).
		 */
		pagefault_out_of_memory();
		return 0;
	}

	if (fault & VM_FAULT_SIGBUS) {
		/*
		 * We had some memory, but were unable to successfully fix up
		 * this page fault.
		 */
		sig = SIGBUS;
		code = BUS_ADRERR;
	} else {
		/*
		 * Something tried to access memory that isn't in our memory
		 * map.
		 */
		sig = SIGSEGV;
		code = fault == VM_FAULT_BADACCESS ?
			SEGV_ACCERR : SEGV_MAPERR;
	}

	__do_user_fault(tsk, addr, esr, sig, code, regs);
	return 0;

no_context:
	__do_kernel_fault(mm, addr, esr, regs);
	return 0;
}
```

### __do_page_fault 

``` c++
/*
 * 1. 通过find_vma查找虚拟地址addr后的最近vma，如果没找到，则方位地址错误，因为它不在所分配的任何一个vma线性区；
 * 2. 如果找到vma，但addr并未落入这个区间，则可能是栈中vma
 * 3. 经检查一切正常后，调用handle_mm_fault分配物理页框
 * VM_FAULT_BADACCESS：严重错误，内核会直接kill该进程
 */
static int __do_page_fault(struct mm_struct *mm, unsigned long addr,
			   unsigned int mm_flags, unsigned long vm_flags,
			   struct task_struct *tsk)
{
	struct vm_area_struct *vma;
	int fault;
	 /* 搜索出现异常的地址前向最近的的vma */
	vma = find_vma(mm, addr);
	fault = VM_FAULT_BADMAP;
	/* 如果vma为NULL，说明addr之后没有vma，所以这个addr是个错误地址 */
	if (unlikely(!vma))
		goto out;
	 /*如果addr之后有vma，但不包含addr，不能断定addr是错误地址，还需检查*/
	if (unlikely(vma->vm_start > addr))
		goto check_stack;

	/*
	 * Ok, we have a good vm_area for this memory access, so we can handle
	 * it.
	 */
good_area:
	/*
	 * Check that the permissions on the VMA allow for the fault which
	 * occurred. If we encountered a write or exec fault, we must have
	 * appropriate permissions, otherwise we allow any permission.
	 */
	if (!(vma->vm_flags & vm_flags)) {
		fault = VM_FAULT_BADACCESS;
		goto out;
	}
	 //分配新页框
	return handle_mm_fault(mm, vma, addr & PAGE_MASK, mm_flags);

check_stack:
	/* 
   	 * addr后面的vma的vm_flags含有VM_GROWSDOWN标志，说明这个vma属于栈的vma
	 * 即addr在栈中，有可能是栈空间不够时再进栈导致的访问错误
	 * 同时检查栈是否还能扩展，如果不能扩展则确认确实是栈溢出导致，即addr确实是栈中地址，不是非法地址
	 * 应进入缺页中断请求
	 */
	if (vma->vm_flags & VM_GROWSDOWN && !expand_stack(vma, addr))
		goto good_area;
out:
	return fault;
}
```

### handle_mm_fault 

``` c++
/*
 * 核心功能封装在__handle_mm_fault中：为引发缺页的进程分配一个物理页框；
 */
int handle_mm_fault(struct mm_struct *mm, struct vm_area_struct *vma,
		    unsigned long address, unsigned int flags)
{
	int ret;

	__set_current_state(TASK_RUNNING);

	count_vm_event(PGFAULT);
	mem_cgroup_count_vm_event(mm, PGFAULT);

	/* do counter updates before entering really critical section. */
	check_sync_rss_stat(current);

	/*
	 * Enable the memcg OOM handling for faults triggered in user
	 * space.  Kernel faults are handled more gracefully.
	 */
	if (flags & FAULT_FLAG_USER)
		mem_cgroup_oom_enable();

	ret = __handle_mm_fault(mm, vma, address, flags);

	if (flags & FAULT_FLAG_USER) {
		mem_cgroup_oom_disable();
                /*
                 * The task may have entered a memcg OOM situation but
                 * if the allocation error was handled gracefully (no
                 * VM_FAULT_OOM), there is no need to kill anything.
                 * Just clean up the OOM state peacefully.
                 */
                if (task_in_memcg_oom(current) && !(ret & VM_FAULT_OOM))
                        mem_cgroup_oom_synchronize(false);
	}

	return ret;
}
```

### brk 

``` c++
/*
 * By the time we get here, we already hold the mm semaphore
 *
 * The mmap_sem may have been released depending on flags and our
 * return value.  See filemap_fault() and __lock_page_or_retry().
 */
static int __handle_mm_fault(struct mm_struct *mm, struct vm_area_struct *vma,
			     unsigned long address, unsigned int flags)
{
	pgd_t *pgd;
	pud_t *pud;
	pmd_t *pmd;
	pte_t *pte;
	/* 如果开启了巨页，则使用hugetlb_fault分配内存 */
	if (unlikely(is_vm_hugetlb_page(vma)))
		return hugetlb_fault(mm, vma, address, flags);
	/* 返回addr对应的一级页表条目 */
	pgd = pgd_offset(mm, address);
	pud = pud_alloc(mm, pgd, address);
	if (!pud)
		return VM_FAULT_OOM;
	pmd = pmd_alloc(mm, pud, address);
	if (!pmd)
		return VM_FAULT_OOM;
	/* 以下这段和huge page有关内容并未能理解?
	 * huge page最小页为2M，相当于PTE=PMD
	 */
	if (pmd_none(*pmd) && transparent_hugepage_enabled(vma)) {
		int ret = VM_FAULT_FALLBACK;
		if (!vma->vm_ops)
			ret = do_huge_pmd_anonymous_page(mm, vma, address,
					pmd, flags);
		if (!(ret & VM_FAULT_FALLBACK))
			return ret;
	} else {
		pmd_t orig_pmd = *pmd;
		int ret;

		barrier();
		if (pmd_trans_huge(orig_pmd)) {
			unsigned int dirty = flags & FAULT_FLAG_WRITE;

			/*
			 * If the pmd is splitting, return and retry the
			 * the fault.  Alternative: wait until the split
			 * is done, and goto retry.
			 */
			if (pmd_trans_splitting(orig_pmd))
				return 0;

			if (pmd_protnone(orig_pmd))
				return do_huge_pmd_numa_page(mm, vma, address,
							     orig_pmd, pmd);

			if (dirty && !pmd_write(orig_pmd)) {
				ret = do_huge_pmd_wp_page(mm, vma, address, pmd,
							  orig_pmd);
				if (!(ret & VM_FAULT_FALLBACK))
					return ret;
			} else {
				huge_pmd_set_accessed(mm, vma, address, pmd,
						      orig_pmd, dirty);
				return 0;
			}
		}
	}

	/*
	 * Use __pte_alloc instead of pte_alloc_map, because we can't
	 * run pte_offset_map on the pmd, if an huge pmd could
	 * materialize from under us from a different thread.
	 */
	if (unlikely(pmd_none(*pmd)) &&
	    unlikely(__pte_alloc(mm, vma, pmd, address)))
		return VM_FAULT_OOM;
	/* if an huge pmd materialized from under us just retry later */
	if (unlikely(pmd_trans_huge(*pmd)))
		return 0;
	/*
	 * A regular pmd is established and it can't morph into a huge pmd
	 * from under us anymore at this point because we hold the mmap_sem
	 * read mode and khugepaged takes it in write mode. So now it's
	 * safe to run pte_offset_map().
	 */
	pte = pte_offset_map(pmd, address);

	return handle_pte_fault(mm, vma, address, pte, pmd, flags);
}
```

### handle_pte_fault 

``` c++

/*
 * 根据页表项pte所描述的物理页框是否在物理内存中，分为两大类，由pte_present(*pte)区分
 * 1. 调页请求(物理页不存在)：分配一个页框，根据pte页表项是否为空分为两种情况：
 * 		a. pte为空：pte中尚未写入物理地址，文件映射缺页中断、匿名映射缺页中断
 * 		b. pte不为空，但(PTE_VALID | PTE_PROT_NONE)未置位：相关物理地址已经被交换到外存
 * 2. 写时复制(物理页存在)：被访问的页存在，但该页是只读，内核需要对该页进行写操作；则此时内核将这个已存在的只读页中数据复制到一个新页框中
 */
static int handle_pte_fault(struct mm_struct *mm,
		     struct vm_area_struct *vma, unsigned long address,
		     pte_t *pte, pmd_t *pmd, unsigned int flags)
{
	pte_t entry;
	spinlock_t *ptl;

	/*
	 * 
	 */
	entry = *pte;
	barrier();
	/* 
	 * pte_present(*pte)==0说明pte实际指向的物理地址不存在，可能是调页请求
	 * 有两种情况：
	 * 1. *pte为空；
	 * 2. *pte不为空，但是(PTE_VALID | PTE_PROT_NONE)标志位未置位
	 */
	if (!pte_present(entry)) {

		if (pte_none(entry)) {
		/* 第一种情况：*pte == 0, 说明尚未写入任何物理地址 */
			if (vma->vm_ops)
			/* 如果该vma定义了操作函数集合，说明是文件映射页面缺页中断，将调用do_fault分配物理页 */
				return do_fault(mm, vma, address, pte, pmd,
						flags, entry);
			/* 匿名映射缺页中断，最终会调用alloc_pages从伙伴系统分配页面 */
			return do_anonymous_page(mm, vma, address, pte, pmd,
					flags);
		}
		/* 
		 * 第二种情况：*pte不为空，但是(PTE_VALID | PTE_PROT_NONE)标志位未置位，即物理页不存在
		 * 此刻该物理页已经由主存换出到了外存，则调用do_swap_page()完成页框分配
		 */
		return do_swap_page(mm, vma, address,
					pte, pmd, flags, entry);
	}
/*=============================以下为物理页面存在场景===============================================*/
	if (pte_protnone(entry))
		return do_numa_page(mm, vma, address, entry, pte, pmd);
	/* 到这应该是写时复制触发的缺页中断；即被访问的页面不可写，有两种情况：
	 * 1. 之前给vma映射的是零页(zero-pfn)
	 * 2. 访问fork得到的进程空间(子进程、父进程共享父进程的内存，均为只读页)
	 */
	ptl = pte_lockptr(mm, pmd);
	spin_lock(ptl);
	if (unlikely(!pte_same(*pte, entry)))
		goto unlock;
	/* 写操作时发生的缺页异常 */
	if (flags & FAULT_FLAG_WRITE) {
		if (!pte_write(entry))/* 第二种情况：pte页表项标识不可写，引发COW */
			return do_wp_page(mm, vma, address,
					pte, pmd, ptl, entry);/* 进行写时复制操作 */
		/* 第一种情况: vma映射的是zero-pfn，设置pte_dirty位，标识页内容已被修改 */
		entry = pte_mkdirty(entry);
	}
	entry = pte_mkyoung(entry);/* 设置访问位，通常是_PAGE_ACCESSD */
	if (ptep_set_access_flags(vma, address, pte, entry, flags & FAULT_FLAG_WRITE)) {
		/* pte内容发生变化，需要把新的内容写入pte页表项中，并刷新TLB和cache */
		update_mmu_cache(vma, address, pte);
	} else {
		/*
		 * This is needed only for protection faults but the arch code
		 * is not yet telling us if this is a protection fault or not.
		 * This still avoids useless tlb flushes for .text page faults
		 * with threads.
		 */
		if (flags & FAULT_FLAG_WRITE)
			flush_tlb_fix_spurious_fault(vma, address);
	}
unlock:
	pte_unmap_unlock(pte, ptl);
	return 0;
}
```

### do_fault 

``` c++
/*
 * 文件映射缺页中断处理函数
 */
static int do_fault(struct mm_struct *mm, struct vm_area_struct *vma,
		unsigned long address, pte_t *page_table, pmd_t *pmd,
		unsigned int flags, pte_t orig_pte)
{
	pgoff_t pgoff = (((address & PAGE_MASK)
			- vma->vm_start) >> PAGE_SHIFT) + vma->vm_pgoff;

	pte_unmap(page_table);
	/* The VMA was not fully populated on mmap() or missing VM_DONTEXPAND */
	if (!vma->vm_ops->fault)
		return VM_FAULT_SIGBUS;
	if (!(flags & FAULT_FLAG_WRITE))
		return do_read_fault(mm, vma, address, pmd, pgoff, flags,
				orig_pte);
	if (!(vma->vm_flags & VM_SHARED))
		return do_cow_fault(mm, vma, address, pmd, pgoff, flags,
				orig_pte);
	return do_shared_fault(mm, vma, address, pmd, pgoff, flags, orig_pte);
}
```

### do_anonymous_page 

``` c++
/*
 * We enter with non-exclusive mmap_sem (to exclude vma changes,
 * but allow concurrent faults), and pte mapped but not yet locked.
 * We return with mmap_sem still held, but pte unmapped and unlocked.
 */
static int do_anonymous_page(struct mm_struct *mm, struct vm_area_struct *vma,
		unsigned long address, pte_t *page_table, pmd_t *pmd,
		unsigned int flags)
{
	struct mem_cgroup *memcg;
	struct page *page;
	spinlock_t *ptl;
	pte_t entry;

	pte_unmap(page_table);

	/* File mapping without ->vm_ops ? */
	if (vma->vm_flags & VM_SHARED)
		return VM_FAULT_SIGBUS;

	/* Check if we need to add a guard page to the stack */
	if (check_stack_guard_page(vma, address) < 0)
		return VM_FAULT_SIGSEGV;

	/* 如果不是写操作，则把zero_pfn的页表条目赋给entry
   	 * 因为这里已经是缺页异常的请求调页的处理，又是读操作，所以肯定是本进程第一次访问这个页，
	 * 所以这个页里面是什么内容无所谓，分配个默认全零页就好，进一步推迟物理页的分配，这就会让entry带着zero_pfn跳到标号setpte
	 */
	if (!(flags & FAULT_FLAG_WRITE) && !mm_forbids_zeropage(mm)) {
		entry = pte_mkspecial(pfn_pte(my_zero_pfn(address),
						vma->vm_page_prot));
		page_table = pte_offset_map_lock(mm, pmd, address, &ptl);
		if (!pte_none(*page_table))
			goto unlock;
		goto setpte;
	}

	/* Allocate our own private page. */
	if (unlikely(anon_vma_prepare(vma)))
		goto oom;
	page = alloc_zeroed_user_highpage_movable(vma, address);
	if (!page)
		goto oom;
	/*
	 * The memory barrier inside __SetPageUptodate makes sure that
	 * preceeding stores to the page contents become visible before
	 * the set_pte_at() write.
	 */
	__SetPageUptodate(page);

	if (mem_cgroup_try_charge(page, mm, GFP_KERNEL, &memcg))
		goto oom_free_page;

	entry = mk_pte(page, vma->vm_page_prot);
	if (vma->vm_flags & VM_WRITE)
		entry = pte_mkwrite(pte_mkdirty(entry));

	page_table = pte_offset_map_lock(mm, pmd, address, &ptl);
	if (!pte_none(*page_table))
		goto release;

	inc_mm_counter_fast(mm, MM_ANONPAGES);
	page_add_new_anon_rmap(page, vma, address);
	mem_cgroup_commit_charge(page, memcg, false);
	lru_cache_add_active_or_unevictable(page, vma);
setpte:
	set_pte_at(mm, address, page_table, entry);

	/* No need to invalidate - it was non-present before */
	update_mmu_cache(vma, address, page_table);
unlock:
	pte_unmap_unlock(page_table, ptl);
	return 0;
release:
	mem_cgroup_cancel_charge(page, memcg);
	page_cache_release(page);
	goto unlock;
oom_free_page:
	page_cache_release(page);
oom:
	return VM_FAULT_OOM;
}
```


## 附录

### pte 信息

``` c++
* _PAGE_PRESENT 指定了虚拟内存页是否存在于内存之中，这个之前的pte_present函数里有使用

* _PAGE_ACCESS CPU每次访问内存页时，会自动设置

* _PAGE_DIRTY表示页是否是脏的，即页的内容是否修改过

* _PAGE_FILE与_PAGE_DIRTY相同，但用于不同的上下文，即页不在内存中的时候

* _PAGE_USER，如果设置了_PAGE_USER则允许用户访问该页，否则，只有内核能够访问

* _PAGE_READ、_PAGE_WRITE、_PAGE_EXECUTE制定了普通的用户进程是否允许读取、写入、执行该页中的机器代码

* 对应于这些标志，内核提供了一些函数来查看和设置不同标志的状态

* pte_present 页是否在内存中

* pte_read 从用户空间是否可以读取该页

* pte_write 是否可以写入该页

* pte_exec 该页中的数据是否可以作为二进制代码执行

* pte_dirty 页的内容是否被修改过

* pte_file 该页表项是否属于非线性映射

* pte_young 访问位(_PAGE_ACCESS)是否设置了

* pte_rdpprote 清除该页的读权限

* pte_wrprote 清除该页的写权限

* pte_exprote 清除该页的二进制数据的权限

* pte_mkread 设置读权限

* pte_mkwrite 设置写权限

* pte_mkexec 允许执行页的内容

* pte_mkdirty 将页标记为脏

* pte_mkclean "清除"页，通常是清除_PAGE_DIRTY位

* pte_mkyoung 设置访问位，通常是_PAGE_ACCESSD

* pte_mkold 清除访问位

* install_special_mapping
```

	
## 参考资料

[内存分配函数总结](https://www.cnblogs.com/arnoldlu/p/8251333.html)

[缺页中断详解](https://blog.csdn.net/u010246947/article/details/10431149)

[COW分析](https://yq.aliyun.com/articles/378695)
