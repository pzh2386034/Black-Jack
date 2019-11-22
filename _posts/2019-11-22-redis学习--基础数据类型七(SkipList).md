---
title: redis学习--基础数据类型七(SkipList)
date: 2019-11-18
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

### zskiplist基本原理

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
/* zskiplist插入节点 */
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    serverAssert(!isnan(score));
    x = zsl->header;
	/* 从跳表的目前最高层网下，在每一层中找到新节点的插入位置 */
    for (i = zsl->level-1; i >= 0; i--) {
        /* 获取待插入节点的位置 */
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    /* we assume the element is not already inside, since we allow duplicated
     * scores, reinserting the same element should never happen since the
     * caller of zslInsert() should test in the hash table if the element is
     * already inside or not. */
    level = zslRandomLevel();
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }
    x = zslCreateNode(level,score,ele);
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        /* update span covered by update[i] as x is inserted here */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    /* increment span for untouched levels */
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}
```



## 附录

### 关键数据结构 

``` c++
/* 有序集合跳跃表节点结构 */
typedef struct zskiplistNode {
    sds ele;
    double score;/* 节点分值，跳跃表中的所有节点都按照分值从小到大排序 */
    struct zskiplistNode *backward;/* 下置节点 */
    struct zskiplistLevel {
        struct zskiplistNode *forward;/* 本node在该levle的前向节点 */
        unsigned long span;/* 该层跨越的节点数量 */
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

[跳跃表原理解释](https://blog.csdn.net/qpzkobe/article/details/80056807)
