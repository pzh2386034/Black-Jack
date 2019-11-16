---
title: redis学习--基础数据类型四(dict)
date: 2019-11-11
categories:
- redis
tags:
- dict 
---


## 前沿

>在上一篇中，我们着重分析了intset整形数据集, 该数据集是有序非重复的，分析了其基本新增数据操作，其中比较重要的数扩容操作;本篇中学习下一种数据类型

### dict简述

* redis中dict为一个hash表，表中每一对key-value为dictentry; 它有一个数据管理结构体，管理所有的dictentry

* redis的dict hash table为解决哈希冲突问题，使用链式地址法

### rehash

>为了保证hash table的负载因子维持在一个合理范围内，当hash table保存的键值对太多或者太少时，redis对hash table大小进行相应的扩展和收缩

* 负载因子 = hash table已保存节点数量/hash table size

* 负载因子越大，意味着hash table越满，越容易hash冲突，性能会降低

* 负载因子越小，意味着hash table越稀疏，浪费了较多内存

### 渐进式rehash

>如果需要rehash的键值对较多，会对服务器造成性能影响；因此引入渐进式rehash

* 渐进式rehash使用了dict结构体中的rehashidx属性辅助完成；

* 当开始时，rehashidx会被设置为0，表示从dictEntry[0]开始进行rehash，每完成一次rehashidx++

* 直到ht[0]中的所有节点都被rehash到ht[1],rehashidx被设置为-1，表示rehash结束

### 本篇主要内容

>分析dict字典类型底层数据结构，以及配套的重要操作函数

## 代码分析

### dict原始数据结构

``` c++
/* dict中的原始key-value数据结构;可以看到value可以是任意数据类型 */
typedef struct dictEntry {
    void *key;/* 键名 */
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;/* 指向下一个节点, 将多个哈希值相同的键值对连接起来*/
} dictEntry;
/* redis字典创建是需要定义数据操作的dictype对象 */
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);         /* 哈希函数 */
    void *(*keyDup)(void *privdata, const void *key);  /* 复制key函数 */
    void *(*valDup)(void *privdata, const void *obj);  /* 复制value函数 */
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);  /* 比较键函数 */
    void (*keyDestructor)(void *privdata, void *key);//销毁key时调用
    void (*valDestructor)(void *privdata, void *obj);//销毁value时调用
} dictType;
/* dict中dictentry hash表 */
typedef struct dictht {
    dictEntry **table;/* 哈希表节点数组 */
    unsigned long size;/* 哈希表大小 */
    unsigned long sizemask;/* 哈希表大小掩码，用于计算哈希表的索引值，大小总是dictht.size-1 */
    unsigned long used;/*哈希表已使用的节点数量 */
} dictht
/* 由dict组成的hash表 */
typedef struct dict {
    dictType *type;/* 类型特定函数 */
    void *privdata;/* 一般为NULL，如果希望通过一些方法将关心的数据透传出去则可使用 */
    dictht ht[2];/* 保存的两个hash table，ht[0]是真正使用的，ht[1]会在rehash时使用 */
    long rehashidx; /* rehashing not in progress if rehashidx == -1，rehash进度节点，如果不等于-1，说明还在进行rehash */
    unsigned long iterators; /* number of iterators currently running，正在运行中的遍历器数量 */
} dict;

/* 迭代dict中dictentry. */
typedef struct dictIterator {
    dict *d;
    long index;
    int table, safe;
    dictEntry *entry, *nextEntry;
    /* unsafe iterator fingerprint for misuse detection. */
    long long fingerprint;
} dictIterator
```

### 创建dict

``` c++
dict *dictCreate(dictType *type,
        void *privDataPtr)
{
	/* 申请dict管理结构体内存. */
    dict *d = zmalloc(sizeof(*d));

    _dictInit(d,type,privDataPtr);
    return d;
}
/* 初始化dict中成员变量. */
int _dictInit(dict *d, dictType *type,
        void *privDataPtr)
{
    _dictReset(&d->ht[0]);
    _dictReset(&d->ht[1]);
    d->type = type;
    d->privdata = privDataPtr;
    d->rehashidx = -1;
    d->iterators = 0;
    return DICT_OK;
}
```

### 释放字典 dictRelease

``` c++
int _dictClear(dict *d, dictht *ht, void(callback)(void *)) {
    unsigned long i;

    /* Free all the elements */
    for (i = 0; i < ht->size && ht->used > 0; i++) {
        dictEntry *he, *nextHe;

        if (callback && (i & 65535) == 0) callback(d->privdata);

        if ((he = ht->table[i]) == NULL) continue;
        while(he) {
            nextHe = he->next;
            dictFreeKey(d, he);
            dictFreeVal(d, he);
            zfree(he);
            ht->used--;
            he = nextHe;
        }
    }
    /* Free the table and the allocated cache structure */
    zfree(ht->table);
    /* Re-initialize the table */
    _dictReset(ht);
    return DICT_OK; /* never fails */
}

/* Clear & Release the hash table */
void dictRelease(dict *d)
{
    _dictClear(d,&d->ht[0],NULL);
    _dictClear(d,&d->ht[1],NULL);
    zfree(d);
}
```

### dictResh

``` c++
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    if (!dictIsRehashing(d)) return 0;/* 如果正在rehash则退出. */

    while(n-- && d->ht[0].used != 0) {/* used表示该hash table中key-value对节点数量，不为0说明还有节点未转移. */
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
		/* 如果该bucket中没有key-value对，则什么都不做，继续下一个bucket; 相当于去掉了无效hash bucket. */
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx];
        /* 将一个bucket中所有key-value节点转移到临时ht[1] */
        while(de) {
            uint64_t h;

            nextde = de->next;
            /* Get the index in the new hash table */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
			/* 使用头插法将元素插入新hash table；降低时间复杂度 */
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;/* 将hucket指向下一个 */
    }

    /* 如果已经完成rehash，则释放旧hash table，转移ht[1]->ht[0] */
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);/* 初始化ht[1] */
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```


## 参考资料

[redis dict源码解析](https://blog.csdn.net/breaksoftware/article/details/53485416)

[腾讯云 dict基本操作解析](https://cloud.tencent.com/developer/article/1383803)

[dict字典的实现](https://www.cnblogs.com/hoohack/p/8241665.html)

[dict数据结构图示](https://segmentfault.com/a/1190000019967687?utm_source=tag-newest)
