---
title: redis学习--基础数据类型六(quicklist)
date: 2019-11-18
categories:
- redis
tags:
- quicklist 
---

## 前沿

>在上一篇中，我们着重分析了ziplist链表的底层数据结构；除此之外redis还自己定义了双端链表adlist，这个和一般的双向无环链表差别不大，就不额外分析了；本篇主要学习quicklist

### list对比

* adlist在插入、删除节点的复杂度低；但是内存利用率低(两个指针就去掉了至少8bytes空间)，且每个entry内存不连续，容易产生内存碎片

* ziplist内存连续，存储效率高，但是插入、删除成本较高，需要频繁进行内存数据搬移、释放、申请内存

* quicklist则通过将每个压缩表用双向链表方式链接起来

### 本篇主要内容


## 代码分析

### ziplistNew

``` c++
/* 在quicklist头、尾节点塞入数据 */
void quicklistPush(quicklist *quicklist, void *value, const size_t sz,
                   int where) {
    if (where == QUICKLIST_HEAD) {
        quicklistPushHead(quicklist, value, sz);
    } else if (where == QUICKLIST_TAIL) {
        quicklistPushTail(quicklist, value, sz);
    }
}
/* 将新value插入quicklist，并更新统计数据
 * 先通过_quicklistNodeAllowInsert检测要插入node的ziplist是否可以容纳本value
 * 如果node已满，则需要新创建node，将value插入node后，再将node放入quicklist相应位置；随后触发quicklist压缩
 * 更新统计数据
 */
int quicklistPushHead(quicklist *quicklist, void *value, size_t sz) {
    quicklistNode *orig_head = quicklist->head;
    if (likely(
            _quicklistNodeAllowInsert(quicklist->head, quicklist->fill, sz))) {
		/* 如果在头节点上ziplist大小没有超过限制，那么新数据被插入到头节点的ziplist中
		 * 通过调用ziplistpush函数完成节点插入；这在上一篇文档中有介绍
		 */
        quicklist->head->zl =
            ziplistPush(quicklist->head->zl, value, sz, ZIPLIST_HEAD);
		/* 用ziplist->zllen更新quicklistNode->sz */
        quicklistNodeUpdateSz(quicklist->head);
    } else {
		/* 如果头节点的ziplist已经满载，则新建一个quicklistNode */
        quicklistNode *node = quicklistCreateNode();
		/* 将新数据塞到新quicklistNode中 */
        node->zl = ziplistPush(ziplistNew(), value, sz, ZIPLIST_HEAD);
		/* 用ziplist->zllen更新quicklistNode->sz */
        quicklistNodeUpdateSz(node);
		/* 最后将新Node插入到quicklist链表中；完事后会对检查老节点是否需要压缩 */
        _quicklistInsertNodeBefore(quicklist, quicklist->head, node);
    }
    quicklist->count++;/* quicklist->count代表整个quicklist中entry的总个数 */
    quicklist->head->count++;/* quicklistNode->count代表本node中entry的总个数 */
    return (orig_head != quicklist->head);
}
/* 检测新数据是否能插入本quicklistNode
 * 首先计算加入新value后，本node的ziplist所需的全部内存空间
 * 检查是否超标：根据fill正负数分为两种检查模式
 *	 1. fill>0,则检查node->count个数是否超标，且ziplist大小不能超8K
 *	 2. fill<0,fill代表node中的ziplist的最大size，只要总size不超标即可
 */
REDIS_STATIC int _quicklistNodeAllowInsert(const quicklistNode *node,
                                           const int fill, const size_t sz) {
    if (unlikely(!node))
        return 0;

    int ziplist_overhead;
    /* 根据ziplist的定义，计算储存prevlen所需的内存大小 */
    if (sz < 254)
        ziplist_overhead = 1;
    else
        ziplist_overhead = 5;

    /* 计算编码encoding需要的内存大小 */
    if (sz < 64)
        ziplist_overhead += 1;
    else if (likely(sz < 16384))
        ziplist_overhead += 2;
    else
        ziplist_overhead += 5;
	/* new_sz只是新增本entry后，ziplist最大占内存大小 */
    /* new_sz overestimates if 'sz' encodes to an integer type */
    unsigned int new_sz = node->sz + sz + ziplist_overhead;
	/* 检查是否超标：根据fill正负数分为两种检查模式
	 * 1. fill>0,则检查node->count个数是否超标，且ziplist大小不能超8K
	 * 2. fill<0,fill代表node中的ziplist的最大size，只要总size不超标即可
	 */
    if (likely(_quicklistNodeSizeMeetsOptimizationRequirement(new_sz, fill)))
        return 1;
    else if (!sizeMeetsSafetyLimit(new_sz))/* 判断new_sz大小是否超标 */
        return 0;
    else if ((int)node->count < fill)/* 判断个数是否超标 */
        return 1;
    else
        return 0;
}
```



### quicklistCompress

>在新插入entry、替换entry、ziplist合并等场景会调用quicklist压缩

``` c++
/* 压缩有两种情况
 * 1. 对单个node中的ziplist压缩
 * 2. 对整个quicklist根据压缩深度(quicklist->compress)遍历压缩所有需要压缩而没有压缩的node
 * 2是1的更广泛情况；实际真实的压缩步骤是在1中
 */
#define quicklistCompress(_ql, _node)                                          \
    do {                                                                       \
        if ((_node)->recompress)                                               \
            quicklistCompressNode((_node));                                    \
        else                                                                   \
            __quicklistCompress((_ql), (_node));                               \
    } while (0)
	
/* 压缩单个node：
 * 1. 申请quicklistZLF结构体，并申请足够的空间用于存放压缩后数据
 * 2. 调用lzf_compress对ziplist进行压缩
 * 3. 根据压缩后实际size调整quicklistZLF大小
 * 4. 最后释放ziplist内存，调整node管理结构体中相关参数
 */
REDIS_STATIC int __quicklistCompressNode(quicklistNode *node) {
#ifdef REDIS_TEST
    node->attempted_compress = 1;
#endif

    /* 如果本节点ziplist太小，蠢货才会去压缩 */
    if (node->sz < MIN_COMPRESS_BYTES)
        return 0;
	/* 申请quicklistLZF节点；按最坏的情况申请足够的内存 */
    quicklistLZF *lzf = zmalloc(sizeof(*lzf) + node->sz);

    
    if (((lzf->sz = lzf_compress(node->zl, node->sz, lzf->compressed,
                                 node->sz)) == 0) ||
        lzf->sz + MIN_COMPRESS_IMPROVE >= node->sz) {
	/* 如果压缩失败、压缩后size没有缩小则不压缩 */
        /* lzf_compress aborts/rejects compression if value not compressable. */
        zfree(lzf);
        return 0;
    }
	/* 根据压缩后的真实size调整内存 */
    lzf = zrealloc(lzf, sizeof(*lzf) + lzf->sz);
	/* 释放原来的ziplist，node->zl指向quicklistLZF */
    zfree(node->zl);
    node->zl = (unsigned char *)lzf;
	/* 置node的模式为LZF */
    node->encoding = QUICKLIST_NODE_ENCODING_LZF;
    node->recompress = 0;
    return 1;
}
```

### quicklistIndex

``` c++
/* idx：要查找的entry的顺序号；根据符号来判断查找方向
 * 由于idx是entry的顺序号，查找过程就很直白了；最终获得一个完成的被查找entry结构体
 * 1. 首先找到属于哪个node，及在该node的ziplist中的偏移量
 * 2. 对node进行解压缩
 * 3. 根据offset，定位entry->zi
 * 4. 获取该节点的value
 * Returns 1 if element found
 * Returns 0 if element not found */
int quicklistIndex(const quicklist *quicklist, const long long idx,
                   quicklistEntry *entry) {
    quicklistNode *n;
    unsigned long long accum = 0;
    unsigned long long index;
    int forward = idx < 0 ? 0 : 1; /* < 0 -> reverse, 0+ -> forward */

    initEntry(entry);
    entry->quicklist = quicklist;

    if (!forward) {
        index = (-idx) - 1;
        n = quicklist->tail;
    } else {
        index = idx;
        n = quicklist->head;
    }

    if (index >= quicklist->count)
        return 0;

    while (likely(n)) {
        if ((accum + n->count) > index) {
            break;
        } else {
            D("Skipping over (%p) %u at accum %lld", (void *)n, n->count,
              accum);
            accum += n->count;
            n = forward ? n->next : n->prev;
        }
    }

    if (!n)
        return 0;

    D("Found node: %p at accum %llu, idx %llu, sub+ %llu, sub- %llu", (void *)n,
      accum, index, index - accum, (-index) - 1 + accum);

    entry->node = n;/* 找到entry所在的node，继续定位offset */
    if (forward) {
        /* forward = normal head-to-tail offset. */
        entry->offset = index - accum;
    } else {
        /* reverse = need negative offset for tail-to-head, so undo
         * the result of the original if (index < 0) above. */
        entry->offset = (-index) - 1 + accum;
    }

    quicklistDecompressNodeForUse(entry->node);/* 需要解压就解压 */
    entry->zi = ziplistIndex(entry->node->zl, entry->offset);/* 根据offset，找到entry首地址 */
    ziplistGet(entry->zi, &entry->value, &entry->sz, &entry->longval);/* 获取entry的content，并反编码 */
    /* The caller will use our result, so we don't re-compress here.
     * The caller can recompress or delete the node as needed. */
    return 1;
}
```



## 附录

### quicklistNode

``` c++
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;/* 如果当前节点的数据没有压缩，则指向一个ziplist结构，否则指向quicklistLZF结构 */
    unsigned int sz;             /* ziplist的内存大小(包括zlbytes, zltail, zllen, zlend和各个数据项)，如果被压缩，表示压缩前的ziplist大小 */
    unsigned int count : 16;     /* ziplist中entry个数，相当于zllen */
    unsigned int encoding : 2;   /* 1为ziplist, 2为LZF压缩存储方式 */
    unsigned int container : 2;  /* 预留字段，目前都是2，表示为ziplist */
    unsigned int recompress : 1; /* 如果本node要重新压缩，则至1 */
    unsigned int attempted_compress : 1; /* 节点是否能够被压缩，只用在测试 */
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;

/* 在每个quicklistNode中，quicklistNode->zl要不指向一个ziplist结构，要不指向一个quicklistLZF */
typedef struct quicklistLZF {
    unsigned int sz; /* 压缩后的ziplist大小 */
    char compressed[];/* 柔性数组，存放压缩后的ziplist字节数组 */
} quicklistLZF;
```

### quicklist

``` c++
/* quicklist链表管理结构体 */
typedef struct quicklist {
    quicklistNode *head;		/* 头节点 */
    quicklistNode *tail;		/* 尾节点 */
    unsigned long count;        /* 所有ziplist entry个数的总和 */
    unsigned long len;          /* quicklistNode节点数量 */
	/* list-max-ziplist-size: 单个quicklistNode的大小限制
	 * fill>0: 表示ziplist中最多可以存多少个entry
	 * fill<0: 表示ziplist最大的内存大小
	 */
    int fill : 16;              
    unsigned int compress : 16; /* list-compress-depth: 压缩深度 */
} quicklist;
```

### quicklistEntry 

``` c++
/* ziplist中entry的管理结构体 */
typedef struct quicklistEntry {
    const quicklist *quicklist;/* 指向所属的quicklist指针 */
    quicklistNode *node;/* 指向所属的quicklistNode节点的指针 */
    unsigned char *zi;/* 指向本entry在ziplist的起始位置 */
    unsigned char *value;/* 指向当前ziplist结构的字符串vlaue成员 */
    long long longval;/* 指向当前ziplist结构的整数value成员 */
    unsigned int sz;/* 保存当前ziplist结构的字节数大小 */
    int offset;/* 保存本value相对ziplist的偏移量 */
} quicklistEntry
```


## 参考资料

[quicklist基本操作解析](https://www.jianshu.com/p/2d0f4833470f)

[redis源码解读六--quicklist](http://czrzchao.com/redisSourceQuicklist)

[quicklist结构图解](https://blog.csdn.net/zhaoliang831214/article/details/82054476)

