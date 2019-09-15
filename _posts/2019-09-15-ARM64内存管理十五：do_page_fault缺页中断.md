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

	1. 通过find_vma找到start, end两个地址所在的线性地址区间,判断要释放的区间是否与已存在的vma area有重叠
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
	
	
	
	
	
## 参考资料

[内存分配函数总结](https://www.cnblogs.com/arnoldlu/p/8251333.html)
