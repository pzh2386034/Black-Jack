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

>分析dict字典类型底层数据结构，以及配套的重要操作函数；其中最重要的当然是dict的扩容及缩容

* 扩容：在链表过长影响查找效率时，扩大数组长度以减小链表长度

* 缩容：在bucket过于稀疏(空桶数量过多)时，减小数组长度使得无效数组指针变少，达到节约空间的目的

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
/* 在redis中字典中的hash表也是采用延迟初始化策略:
 * 在创建字典的时候并没有为hash table分配内存，只有当第一次插入数据时，才真正分配内存 */
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

### dictRehash

``` c++
/* 入参：dict 指针和步进长度n
 * 步进长度：每次dictRehash最多rehash的真实节点；对于空的bucket则最多为步进长度10倍
 */
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

### dict扩容

>在链表过长影响查找效率时，扩大数组长度以减小链表长度，达到性能优化

``` c++
/* 两种情况需要调用dicExpand扩容数组：
 * 1. hash table中bucket数量为0,
 * 2. 平均每个bucket中元素数量，如果>5，则要扩容dict
 */
static int _dictExpandIfNeeded(dict *d)
{
    /* Incremental rehashing already in progress. Return. */
    if (dictIsRehashing(d)) return DICT_OK;

    /* 如果哈希表ht[0]的大小为0，则初始化字典. */
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    /* used: hash table中真实元素的数量
	 * size：hash table中bucket数量
	 * used/size: 平均每个bucket中元素数量，如果>5，则要扩容dict
	 * dict_can_resize说明见附录
	 */
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        return dictExpand(d, d->ht[0].used*2);
    }
    return DICT_OK;
}

/* Expand or create the hash table */
int dictExpand(dict *d, unsigned long size)
{
    /* 此size是希望dict扩张到多少个 */
    if (dictIsRehashing(d) || d->ht[0].used > size)
        return DICT_ERR;

    dictht n; /* 新建hash table */
	/* _dictNextPower是获取最近接size的，但是比size大的2的N次幂 */
    unsigned long realsize = _dictNextPower(size);

    /* Rehashing to the same table size is not useful. */
    if (realsize == d->ht[0].size) return DICT_ERR;

    /* 初始化new hash table相关数据 */
    n.size = realsize;
    n.sizemask = realsize-1;
    n.table = zcalloc(realsize*sizeof(dictEntry*));
    n.used = 0;

    /* Is this the first initialization? If so it's not really a rehashing
     * we just set the first hash table so that it can accept keys. */
    if (d->ht[0].table == NULL) {
        d->ht[0] = n;
        return DICT_OK;
    }

    /* Prepare a second hash table for incremental rehashing */
    d->ht[1] = n;
    d->rehashidx = 0;/* 置rehash标志位，即扩展空间后，必定会进行rehash */
    return DICT_OK;
}
```

### 缩容 

>在bucket过于稀疏(空桶数量过多)时，减小数组长度使得无效数组指针变少，达到节约空间的目的

``` c++
/* 当hash table保存的key-value数量与bucket大小比例<10%是缩容，最小容量为4 */
int htNeedsResize(dict *dict) {
    long long size, used;

    size = dictSlots(dict);
    used = dictSize(dict);
    return (size > DICT_HT_INITIAL_SIZE &&
            (used*100/size < HASHTABLE_MIN_FILL));
}

/* 将hash table中bucket数量调整至element元素数量，即used/size=1 */
int dictResize(dict *d)
{
    int minimal;

    if (!dict_can_resize || dictIsRehashing(d)) return DICT_ERR;
    minimal = d->ht[0].used;
    if (minimal < DICT_HT_INITIAL_SIZE)
        minimal = DICT_HT_INITIAL_SIZE;
    return dictExpand(d, minimal);
}
```

### dictAddRaw

``` c++
/* 基础操作函数，如果发现正在进行rehash，则会执行步进为1的rehash
 * 使用头插法将元素插入新hash
 */
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
{
    long index;
    dictEntry *entry;
    dictht *ht;
	/* 类似在dictFind,dictGenericDelete,dictGetRandomKey,dictGetSomeKeys等函数都有判断是否在rehash
     * 如果在rehash且没有安全迭代器(d->iterators == 0)绑定到hash table时，则会执行步进为1的rehash操作. */
    if (dictIsRehashing(d)) _dictRehashStep(d);

    /* 确定新元素要插入到哪个hash bucket，dictHashKey：对新key使用hashfunction，计算hash key */
    if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
        return NULL;

    /* 如果正在rehash，则要将新元素添加到ht[1]中 */
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    entry = zmalloc(sizeof(*entry));/* 申请一个hashentry内存空间 */
	/* 使用头插法将元素插入新hash */
    entry->next = ht->table[index];
    ht->table[index] = entry;
    ht->used++;

    /* Set the hash entry fields. */
    dictSetKey(d, entry, key);
    return entry;
}

static long _dictKeyIndex(dict *d, const void *key, uint64_t hash, dictEntry **existing)
{
    unsigned long idx, table;
    dictEntry *he;
    if (existing) *existing = NULL;

    /* Expand the hash table if needed */
    if (_dictExpandIfNeeded(d) == DICT_ERR)
        return -1;
	/* 在两个hash table中搜索是否有同样key的元素存在 */
    for (table = 0; table <= 1; table++) {
		/* 确定该key应该对应在哪个bucket中 */
        idx = hash & d->ht[table].sizemask;
        /* 在该bucket中搜索是否存在相同的Key */
        he = d->ht[table].table[idx];
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key)) {
                if (existing) *existing = he;
                return -1;/* 如果已经存在，则返回-1，不需要新增 */
            }
            he = he->next;
        }
        if (!dictIsRehashing(d)) break;
    }
    return idx;
}
```

>其它删改查的函数基本类似，没有什么特殊的，就不再一个一个分析了

### 

``` c++
```

## 附录

>在updateDictResizePolicy函数中会更新dict_can_resize全局变量

* 在没有子进程执行aof文件重写或生成RDB文件，则允许rehash

* redis中每次开始执行aof文件重写或者开始生成新的RDB文件或者执行aof重写/生成RDB的子进程结束时，都会调用updateDictResizePolicy函数

``` c++
void updateDictResizePolicy(void) {
    if (server.rdb_child_pid == -1 && server.aof_child_pid == -1)
        dictEnableResize();
    else
        dictDisableResize();
}
  
void dictEnableResize(void) {
    dict_can_resize = 1;
}
  
void dictDisableResize(void) {
    dict_can_resize = 0;
}
```

## 参考资料

[redis dict源码解析](https://blog.csdn.net/breaksoftware/article/details/53485416)

[腾讯云 dict基本操作解析](https://cloud.tencent.com/developer/article/1383803)

[dict字典的实现](https://www.cnblogs.com/hoohack/p/8241665.html)

[dict数据结构图示](https://segmentfault.com/a/1190000019967687?utm_source=tag-newest)

[渐进式rehash机制](https://www.cnblogs.com/williamjie/p/11205593.html)
