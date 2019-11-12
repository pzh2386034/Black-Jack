---
title: redis学习--基础数据类型二(zipmap)
date: 2019-11-11
categories:
- redis
tags:
- zipmap
---

## 前沿

>在上一篇中，我们着重分析了sds字符串类型；了解该类型的底层数据结构，以及如何实现动态内存分配

### 本篇主要内容

>在本篇中，学习下zipmap数据类型，以及对应的一系列基础操作函数(查找、set、get)

* zipmap的内存布局

zmlen|key length|key|value length|free|value|...|end
--|:--:|--:|:--:|--:|:--:|:--:|:--:

* zmelen: 1字节，记录当前zipmap中key-value对的数量；由于zmlen只有1个字节，因此规定其表示的数量只能为0~254，当mzlen>254时，就需要遍历整个zipmap来得到key-value对的个数

* len：用于记录key或value的长度，有两种情况
	
	1. 当len的第一个字节为0~253时，那么len就只占用这一个字节
	
	2. 当len的第一个字节为254时，那么真实len将用后面的4个字节来表示
	
	3. 因此len要么占用1字节，要么占用5字节

* free: 1字节，表示随后的value后面的空闲字节数，这主要是改变key的value引起的，当free的字节数过大用1个字节不足以表示时，zipmap就会重新分配内存，保证字符串尽量紧凑。

* end：1个字节，为0xFF，用于标志zipmap的结束

## 代码分析

### 创建zipmap

``` c++
/* 对于一个空的zipmap只有2个字节，1个字节的zmlen=0,1一个字节的end=0xFF */
unsigned char *zipmapNew(void) {
    unsigned char *zm = zmalloc(2);

    zm[0] = 0; /* Length */
    zm[1] = ZIPMAP_END;
    return zm;
}
```

### zipmapSet

``` c++
/* 在zipmap结构中增加一对key-value.
 * 1. 如果key已经在zipmap中存在，则检测原有的key-value entry总长度能否满足新数据的长度要求
		a. 不行则先将zipmap扩容，将p指针定位到key起始点
 * 2. 如果key在zipmap中不存在，则扩容zipmap，并将p指针指到新扩容的起始位置
 * 3. 检测freedata长度是否超过254，如果是则要收缩zipmap
 * 4. 将新的key，value数据更新到zipmap中，从p指针指向位置开始
 * . */
unsigned char *zipmapSet(unsigned char *zm, unsigned char *key, unsigned int klen, unsigned char *val, unsigned int vlen, int *update) {
    unsigned int zmlen, offset;
	/* 根据klen, vlen确定需要申请的内存大小，默认：1(klen)+1(vlen)+1(free) + klen + vlen */
    unsigned int freelen, reqlen = zipmapRequiredLength(klen,vlen);
    unsigned int empty, vempty;
    unsigned char *p;

    freelen = reqlen;
    if (update) *update = 0;
	/* 遍历zipmap中所有key-value，确定key是否已存在；如果存在则返回key地址偏移量；
	 * 看明白附录中的几个辅助函数再来理解逻辑很清晰 */
    p = zipmapLookupRaw(zm,key,klen,&zmlen);
    if (p == NULL) {
        /* key没有存在，扩充zipmap，容纳新键值对 */
        zm = zipmapResize(zm, zmlen+reqlen);
        p = zm+zmlen-1;
        zmlen = zmlen+reqlen;

        /* 如果zipmap中key-value对数量没有达到254，则递增 */
        if (zm[0] < ZIPMAP_BIGLEN) zm[0]++;
    } else {
        /* Key found. Is there enough space for the new value? */
        /* Compute the total length: */
        if (update) *update = 1;
		/* 计算原有的key-value entry总长度， p指针指向的是key的起始地址 */
        freelen = zipmapRawEntryLength(p);
        if (freelen < reqlen) {
            /* Store the offset of this key within the current zipmap, so
             * it can be resized. Then, move the tail backwards so this
             * pair fits at the current position. */
			 /* 获取该key相对zm结构体首地址偏移量 */
            offset = p-zm;
			/* 如果没有空间插入新的值，则调整大小 */
            zm = zipmapResize(zm, zmlen-freelen+reqlen);
			/* 将p指针定位到重复key起始点 */
            p = zm+offset;

            /* 将p指针后留出reqlen长度空间；把原有的数据向后mov */
            memmove(p+reqlen, p+freelen, zmlen-(offset+freelen+1));
            zmlen = zmlen-freelen+reqlen;/* 更新zmlen长度 */
            freelen = reqlen;/* 更新freelen为新的key-value entry总长度 */
        }
    }

    /* 检测空闲数据区是否超过ZIPMAP_VALUE_MAX_FREE，如果超过则要缩小，使得zipmap空间紧凑 */
    empty = freelen-reqlen;
    if (empty >= ZIPMAP_VALUE_MAX_FREE) {
        /* First, move the tail <empty> bytes to the front, then resize
         * the zipmap to be <empty> bytes smaller. */
        offset = p-zm;
        memmove(p+reqlen, p+freelen, zmlen-(offset+freelen+1));
        zmlen -= empty;
        zm = zipmapResize(zm, zmlen);
        p = zm+offset;
        vempty = 0;
    } else {
        vempty = empty;
    }

    /* 将新的key，value数据更新到zipmap中 */
    /* Key: */
    p += zipmapEncodeLength(p,klen);
    memcpy(p,key,klen);
    p += klen;
    /* Value: */
    p += zipmapEncodeLength(p,vlen);
    *p++ = vempty;
    memcpy(p,val,vlen);
    return zm;
}
```

### zipmapGet

``` c++
/* 获取指定key的value信息
 * value：返回指针指向value data首地址；vlen：返回value data长度
 * 1. zipmapLookupRaw：找到指定key的起始位置
 */
int zipmapGet(unsigned char *zm, unsigned char *key, unsigned int klen, unsigned char **value, unsigned int *vlen) {
    unsigned char *p;
	/* 遍历zipmap中所有key-value，确定key是否已存在；如果存在则返回key地址偏移量；
	 * 看明白附录中的几个辅助函数再来理解逻辑很清晰 */
    if ((p = zipmapLookupRaw(zm,key,klen,NULL)) == NULL) return 0;
    p += zipmapRawKeyLength(p);		/* 将p指针指向value域的起始地址 */
    *vlen = zipmapDecodeLength(p);		/* vlen：指向value length起始地址 */
    *value = p + ZIPMAP_LEN_BYTES(*vlen) + 1; 		/* 将value指针指向value data首地址 */
    return 1;
}
```

### zipmapNext

``` c++
/* 迭代zm中key-value对
 * 1. 如果zm[0]=0xff则结束
 * 2. zm每次定位到一个新的key-value起始地址
 */
unsigned char *zipmapNext(unsigned char *zm, unsigned char **key, unsigned int *klen, unsigned char **value, unsigned int *vlen) {
    if (zm[0] == ZIPMAP_END) return NULL;
    if (key) {
        *key = zm;
        *klen = zipmapDecodeLength(zm);
        *key += ZIPMAP_LEN_BYTES(*klen);
    }
    zm += zipmapRawKeyLength(zm);
    if (value) {
        *value = zm+1;
        *vlen = zipmapDecodeLength(zm);
        *value += ZIPMAP_LEN_BYTES(*vlen);
    }
    zm += zipmapRawValueLength(zm);
    return zm;
}
```

## 附录

### zipmapEncodeLength

>长度信息编码

``` c++
/* 通用函数，计算key，value的长度及在相应位置填入数据
 * 1. 值小于0xFE，则只有8位表示长度，且内容就是长度值
 * 2. 长度大于0xFE，则有40位表示长度信息，其前8位内容是0xFE，后32位是值内容
 */
static unsigned int zipmapEncodeLength(unsigned char *p, unsigned int len) {
    if (p == NULL) {
	/* 如果第一个参数传NULL，根据第二个参数决定需要多长的空间去存储长度信息 */
        return ZIPMAP_LEN_BYTES(len);
    } else {
	/* 如果第一个参数有值，则根据第二个参数决定在该地址的不同偏移位置设置相应的值 */
        if (len < ZIPMAP_BIGLEN) {
            p[0] = len;
            return 1;
        } else {
            p[0] = ZIPMAP_BIGLEN;
            memcpy(p+1,&len,sizeof(len));
            memrev32ifbe(p+1);
            return 1+sizeof(len);
        }
    }
}
```

### zipmapDecodeLength

>长度信息解码，是编码函数的逆向操作

``` c++
/* 通用函数，计算data的长度
 * 判断传入的长度信息起始地址的内容是否小于0xFE。如果是则该8位就是长度值；否则后移8位，之后的32位才是长度值
 */
static unsigned int zipmapDecodeLength(unsigned char *p) {
    unsigned int len = *p;

    if (len < ZIPMAP_BIGLEN) return len;
    memcpy(&len,p+1,sizeof(unsigned int));
    memrev32ifbe(&len);
    return len;
}
```


### zipmapRawKeyLength

``` c++
/* 计算keylen+keydata len长度和
 * 通过获取的KeyData长度计算出KeyLen Struct的长度，然后将两个长度相加
 */
static unsigned int zipmapRawKeyLength(unsigned char *p) {
    unsigned int l = zipmapDecodeLength(p);
    return zipmapEncodeLength(NULL,l) + l;
}
```

### zipmapRawValueLength

``` c++
/* 计算vallen + free len +value data len+free data len长度和
 * 通过获取的KeyData长度计算出KeyLen Struct的长度，然后将两个长度相加
 */
static unsigned int zipmapRawValueLength(unsigned char *p) {
    unsigned int l = zipmapDecodeLength(p);
    unsigned int used;
	/* used=valuelen struct长度, p[used]=free内容，即freedata长度 */
    used = zipmapEncodeLength(NULL,l);
    used += p[used] + 1 + l;//valuelen +=free data len + free len + value date len
    return used;
}
```

### zipmapRawEntryLength

``` c++
/* 计算一对key-value entry的长度 */
static unsigned int zipmapRawEntryLength(unsigned char *p) {
    unsigned int l = zipmapRawKeyLength(p);
	/* 让指针指向value信息首地址，计算vlaue所有信息的长度 */
    return l + zipmapRawValueLength(p+l);
}
```

## 参考资料

[腾讯云-zipmap源码解析](https://cloud.tencent.com/developer/article/1383741)
