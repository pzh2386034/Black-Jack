---
title: redis学习--基础数据类型七(SkipList)
date: 2019-11-23
categories:
- redis
tags:
- SkipList 
---

## 前沿

>在上一篇中，我们着重分析了quicklist的底层数据结构；它其实是双向链表+ziplist，结合两者的优点，扬长避短的结果；本篇主要学习zskiplistNode数据结构及使用方式

### zskiplist目标

* 对于一般的链表，查询一个元素的时间复杂度为O(n)；即使为有序链表，也无法通过二分缩减复杂度，因为链表地址不连续，没法跳跃查询

* 对于性能要求较高的server，需要高效的查询；红黑树、跳跃表都可以达到差不多的效果，时间复杂度为O(n)

* 在更新数据时，跳表需要更新的部分较少；而红黑树有个再平衡过程，期间要锁定整颗树，在高并发场景难以满足条件

* 跳表增删操作更加局部性，不需要锁定整个表，而且过程相对简单清晰

### zskiplist基本性质

* 由很多层结构组成 

* 每一层都是一个有序的单向链表

* 最底层(Level 0)的链表包含所有元素，且为双向链表

* 如果一个元素出现在 Level i 的链表中，则它在 Level i 之下的链表也都会出现


图解比较直观，[参考资料](#参考资料)

### 本篇主要内容

>分析zskiplist底层数据结构及实现方式，分析基本增删操作代码

## 代码分析

### 创建跳表 zslCreateNode 

``` c++
/* 以指定的level创建跳跃表节点.
 * The SDS string 'ele' is referenced by the node after the call. */
zskiplistNode *zslCreateNode(int level, double score, sds ele) {
    zskiplistNode *zn =
        zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
    zn->score = score;
    zn->ele = ele;
    return zn;
}

/* Create a new skiplist. */
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    zsl = zmalloc(sizeof(*zsl));
    zsl->level = 1;/* 初始化时，节点的最大层数只有1 */
    zsl->length = 0;/* 初始化时，跳跃表中没有节点，长度为0 */
	/* 创建表头节点，level为32 */
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
	/* 初始化表头各层 */
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;
    zsl->tail = NULL;
    return zsl;
}
```




### zslInsert 

``` c++
/* zskiplist插入节点：关键在两个数组update[],rank[]
 * 1. 从zskiplist头部开始遍历，找到new ele在每一层插入的位置，记录在update[i]中
 * 2. zslRandomLevel：确定new ele的层数new level；如果 new level>zskiplist->level，则高出的层数要初始化
 * 3. 以传入数据创建新节点，并对new ele每一层插入该new node，更新链表及span
 * 4. 更新new ele的backward指针，zskiplist长度+1
 */
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
	/* update数组存储每一层位于插入节点的前一个节点 */
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
	/* rank[0]+1 == 新节点的排名
	 * 其余都是为计算rank[0]产生的中间变量最终目标是rank[0] */
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    serverAssert(!isnan(score));
    x = zsl->header;
	/* 从跳表的目前最高层网下，在每一层中找到新节点的插入位置 */
    for (i = zsl->level-1; i >= 0; i--) {
        /* 可以根据图形想象下，如何计算new ele的最终排名 */
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
		/* 1. 在本层已经没有后继节点
		 * 2. 如果forward->score< score || score相等，但是元素排序新元素较大
		 * 则继续在本层搜索插入点
		 */
            rank[i] += x->level[i].span;// 记录沿途跨越了多少个节点
            x = x->level[i].forward;/* 检查本层下一个节点 */
        }
        update[i] = x;/* 否则，找到了插入点x，新元素在i层要插在x后面 */
    }
	/* update[i]->forward即为新节点在i层将要插入的位置 */
    /* we assume the element is not already inside, since we allow duplicated
     * scores, reinserting the same element should never happen since the
     * caller of zslInsert() should test in the hash table if the element is
     * already inside or not. */
	/* 确定新节点的层数，由大数定律保证当数据量很大时，每一层的节点数量是其上一层的1/2 */
    level = zslRandomLevel();
	/* 如果计算出来的层数要大于目前zskiplist的最高层数；则要将新增的层数初始化 */
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
			/* 由于update[i]->forward后面要跟new ele，因此要初始化扩展的update数组 */
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }
	/* 以传入的数据创建新节点 */
    x = zslCreateNode(level,score,ele);
    for (i = 0; i < level; i++) {
		/* 在要插入的每一层中，其实就是链表插入动作；
		 * 真实的链表是level数组，只需要更新当前层的链表，for循环完成后自然每一层的链表都更新完了 */
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        /* 计算x在i层新位置的span；插入一个节点，相当于原有的update[i]->level[i].span被截成两段 */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
		/* 计算x在i层前一个节点的span */
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    /* 对于new ele没有触及的高层，要增加节点间的跨度 */
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }
	/*  */
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}
```


### zslDelete

``` c++
/* 根据分数及sds删除节点
 * 1. 首先逐层搜索可能删除节点的位置，记录在update数组中；但是这不是最终要删节点的数组
 * 2. 确定在第0层找到的元素x是否为要删除节点，不是则zskiplist中无该元素
 * 3. 如果找到该元素则zslDeleteNode,逐层删除该节点
 * 4. 是否该节点内存，返回1表示成功
 */
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;
	/* 类似insert，从最高层往下逐层链表寻找要删除的节点(匹配score及ele即可) */
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward;
        }
        update[i] = x;/* 在每一层链表中找到的要删除node，记录在update[i]中 */
    }
    /* We may have multiple elements with the same score, what we need
     * is to find the element with both the right score and object. */
    x = x->level[0].forward;
	/* 第0层的数据是全的，如果第0层都找不到要删除节点，则无该节点 */
    if (x && score == x->score && sdscmp(x->ele,ele) == 0) {
        zslDeleteNode(zsl, x, update);/* 找到节点x，则要逐层删除该节点 */
        if (!node)
            zslFreeNode(x);
        else
            *node = x;
        return 1;
    }
    return 0; /* not found */
}

/* 逐层删除元素x */
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    int i;
    for (i = 0; i < zsl->level; i++) {
        if (update[i]->level[i].forward == x) {
			/* 从zsl最高层开始；如果update[i].forward==x，说明i层存在x节点，要删除
			 * 1. 重新计算span   2. i层单项链表调整
			 */
            update[i]->level[i].span += x->level[i].span - 1;
            update[i]->level[i].forward = x->level[i].forward;
        } else {
		/* i层无x，也要调整span值；因为元素个数少了1个
		 * span：表示记录本节点和其forward节点的距离中间间隔的ele个数 */
            update[i]->level[i].span -= 1;
        }
    }
	/* 调整第0层的backward指针 */
    if (x->level[0].forward) {
        x->level[0].forward->backward = x->backward;
    } else {
        zsl->tail = x->backward;
    }
	/* 确定是否要调整跳跃表最大层数(被删除节点是跳跃表中高层链表的唯一元素) */
    while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
        zsl->level--;
    zsl->length--;
}
```

## 附录

### 关键数据结构 

``` c++
/* 有序集合跳跃表节点结构 */
typedef struct zskiplistNode {
    sds ele;
    double score;/* 节点分值，跳跃表中的所有节点都按照分值从小到大排序 */
    struct zskiplistNode *backward;/* 后向指针；只在第0层使用 */
    struct zskiplistLevel {
        struct zskiplistNode *forward;/* 本node在该levle的前向节点 */
        unsigned long span;/* 跨度，用于记录本节点和其forward节点的距离中间间隔的ele个数 */
    } level[];/* 跳跃表层结构，一个zskiplistLevel数组 */
} zskiplistNode;
/* 有序集合跳跃表结构 */
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;/* 跳跃表头尾节点 */
    unsigned long length;/* 跳跃表长度，即跳跃表当前的节点数量(表头节点不算在内，因为初始化默认创建表头) */
    int level;/* 当前跳跃表中层数最大的节点的层数(表头不算在层数内) */
} zskiplist;

typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset
```


## 参考资料

[跳跃表原理解析](https://blog.csdn.net/qpzkobe/article/details/80056807)

[跳跃表增删节点解析](http://www.voidcn.com/article/p-ybgipasw-bsa.html)
