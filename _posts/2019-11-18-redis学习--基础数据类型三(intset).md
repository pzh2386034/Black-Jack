---
title: redis学习--基础数据类型三(intset)
date: 2019-11-11
categories:
- redis
tags:
- intset
---

## 前沿

>在上一篇中，我们着重分析了zipmap压缩hash, 分析了其基本get，set;本篇中学习下一种数据类型

### 本篇主要内容

>分析intset数据集底层数据结构，以及配套的重要操作函数


### 大小端模式简述

* 大端结构：将数据的高位放在地址的低位，符合人类认知方式

0xff000000|0xff000000|0xff000000|0xff000000
|:--:|--:|:--:|--:
0x00|0x12|0x34|0x56

* 小端结构：将数据的高位放在地址的高位，便于计算机计算(便于加法进位，大端结构加法进位会产生回溯问题)；

0xff000000|0xff000000|0xff000000|0xff000000
|:--:|--:|:--:|--:
0x56|0x34|0x12|0x00

* 对于高性能计算的场景优先使用小端结构；而对于存储和网络交互的场景则优先使用大端结构

* redis作为高性能，高并发的内存数据库使用大端结构保存数据


## 代码分析

### 创建intset

``` c++
/* intset原始数据结构 */
typedef struct intset {
//根据元素数值范围确定编码方式，INTSET_ENC_INT16/32/64
    uint32_t encoding;
    uint32_t length;//表示元素数量
    int8_t contents[];//整型数组首地址
} intset;

/* 通过zmalloc在堆上创建一个intset */
intset *intsetNew(void) {
    intset *is = zmalloc(sizeof(intset));
    is->encoding = intrev32ifbe(INTSET_ENC_INT16);
    is->length = 0;
    return is;
}
```

### 重分配集合空间intsetResize

``` c++
/* intset结构是个可变长度结构，根据需要新增空槽个数来重新分配空间 */
static intset *intsetResize(intset *is, uint32_t len) {
    uint32_t size = len*intrev32ifbe(is->encoding);
    is = zrealloc(is,sizeof(intset)+size);
    return is;
}
```

### 获取集合数据长度intsetLen

``` c++
/* Return intset length */
uint32_t intsetLen(intset *is) {
    return intrev32ifbe(is->length);
}
```

### 获取集合占用空间总大小intsetBlobLen

``` c++
/* Return intset blob size in bytes. */
size_t intsetBlobLen(intset *is) {
    return sizeof(intset)+intrev32ifbe(is->length)*intrev32ifbe(is->encoding);
}
```

### 通过位置设置值_intsetSet

``` c++
/* 通过指定pos，更改intset数据. */
static void _intsetSet(intset *is, int pos, int64_t value) {
    uint32_t encoding = intrev32ifbe(is->encoding);

    if (encoding == INTSET_ENC_INT64) {
	/* 强转指针为encoding指定类型，使指针步进长度为元素类型的长度. */
        ((int64_t*)is->contents)[pos] = value;
		/* 将数据转换成大端存储. */
        memrev64ifbe(((int64_t*)is->contents)+pos);
    } else if (encoding == INTSET_ENC_INT32) {
        ((int32_t*)is->contents)[pos] = value;
        memrev32ifbe(((int32_t*)is->contents)+pos);
    } else {
        ((int16_t*)is->contents)[pos] = value;
        memrev16ifbe(((int16_t*)is->contents)+pos);
    }
}
```

### 通过位置获取值_intsetGetEncoded

``` c++
static int64_t _intsetGetEncoded(intset *is, int pos, uint8_t enc) {
    int64_t v64;
    int32_t v32;
    int16_t v16;

    if (enc == INTSET_ENC_INT64) {
        memcpy(&v64,((int64_t*)is->contents)+pos,sizeof(v64));
        memrev64ifbe(&v64);
        return v64;
    } else if (enc == INTSET_ENC_INT32) {
        memcpy(&v32,((int32_t*)is->contents)+pos,sizeof(v32));
        memrev32ifbe(&v32);
        return v32;
    } else {
        memcpy(&v16,((int16_t*)is->contents)+pos,sizeof(v16));
        memrev16ifbe(&v16);
        return v16;
    }
}

/* Return the value at pos, using the configured encoding. */
static int64_t _intsetGet(intset *is, int pos) {
    return _intsetGetEncoded(is,pos,intrev32ifbe(is->encoding));
}
```

### 搜索元素intsetSearch

``` c++
/* 搜索is结构体，是否存在value值；存在则返回pos下标. 
 * 由于intset是有序数组，可以通过二分查找快速定位，log2n
 */
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
    int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
    int64_t cur = -1;

    /* The value can never be found when the set is empty */
    if (intrev32ifbe(is->length) == 0) {
        if (pos) *pos = 0;
        return 0;
    } else {
        /* Check for the case where we know we cannot find the value,
         * but do know the insert position. */
        if (value > _intsetGet(is,max)) {
            if (pos) *pos = intrev32ifbe(is->length);
            return 0;
        } else if (value < _intsetGet(is,0)) {
            if (pos) *pos = 0;
            return 0;
        }
    }
	/* 二分搜索，快速定位 */
    while(max >= min) {
        mid = ((unsigned int)min + (unsigned int)max) >> 1;
        cur = _intsetGet(is,mid);
        if (value > cur) {
            min = mid+1;
        } else if (value < cur) {
            max = mid-1;
        } else {
            break;
        }
    }

    if (value == cur) {
        if (pos) *pos = mid;
        return 1;
    } else {
        if (pos) *pos = min;
        return 0;
    }
}
```

### 新增一个更大数值类型元素 intsetUpgradeAndAdd

``` c++
/* 往集合中新增元素，涉及数据类型改变，需要对整个结构进行升级. */
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
	/* 先获取之前集合类型及长度；再计算新集合类型*/
    uint8_t curenc = intrev32ifbe(is->encoding);
    uint8_t newenc = _intsetValueEncoding(value);
    int length = intrev32ifbe(is->length);
	/* 检查新增值是否为负数，由于value的绝对值比之前数组中所有元素都要大
	 * 如果为负数：则比之前任何元素都小，则要放在头部
	 * 如果为正数：则放在尾部
	 */
    int prepend = value < 0 ? 1 : 0;

    /* 更新intset结构体中数据编码格式；扩大结构体大小 */
    is->encoding = intrev32ifbe(newenc);
    is = intsetResize(is,intrev32ifbe(is->length)+1);

    /* 通过intsetResize重新分配集合内存后，当前内存数据分布和之前的一致，除了多了一个数据位.
	 * 通过while循环转移数据到应该在的位置；如果prepend=0，则不需要转移，因为value要添加在尾部
	 * 如果prepend = 1，则每个数据都要后移一位，因为value要添加在数据头部
	 */
    while(length--)
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

    /* value为负，则在插在头部. */
    if (prepend)
        _intsetSet(is,0,value);
    else
	/* value为正，则在插在尾部. */
        _intsetSet(is,intrev32ifbe(is->length),value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);/* 更新intset结构体中数据数量 */
    return is;
}
```

### 新增数据通用接口 intsetAdd

``` c++
/* Insert an integer in the intset */
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
	/* 获取新数据的类型 */
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 1;

    /* 如果新数据比is的数据类型大，则需要调用特殊接口完成数据插入 */
    if (valenc > intrev32ifbe(is->encoding)) {
        /* This always succeeds, so we don't need to curry *success. */
        return intsetUpgradeAndAdd(is,value);
    } else {
        /* 否则，先搜索value是否已经在is中，如果已存在，则置插入失败，返回. */
        if (intsetSearch(is,value,&pos)) {
            if (success) *success = 0;
            return is;
        }
		/* 扩张数组，并在pos处空出一个槽位. */
        is = intsetResize(is,intrev32ifbe(is->length)+1);
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }
	/* 在pos出插入数据. */
    _intsetSet(is,pos,value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```

Note：

* 通过上面的函数，我们发现Redis的整数集在增删改元素时要自动调整元素排序;

* 在新增绝对值超过当前集合可以表达的数据时，升级当前集合;

* 但是如果删除元素时，即使现存的数字都比当前集合表达的区间的最小值还要小，也不会发生降级的操作;


## 参考资料

[腾讯云-zipmap源码解析](https://cloud.tencent.com/developer/article/1383740)
