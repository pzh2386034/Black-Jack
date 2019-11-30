---
title: redis学习--基础数据类型八(stream)
date: 2019-11-28
categories:
- redis
tags:
- stream 
---

## 前沿

>在上一篇中，我们着重分析了zskiplist的底层数据结构，zskiplist是一个金字塔形的多层链表；除了level0层的链表是双向的外，其余都是单向链表；今天学习redis5的全新数据类型steams，它是一个新的强大的支持多播的可持久化消息队列，借鉴了kafka的设计

### redis stream内存组织

* stream的底层数据是radix tree，每个node存储了一个listpack

* listpack是一块连续的内存block，用于序列化msg entry及相关元信息，如msg ID，使用了多种编码，用于节省内存，是ziplist的升级版

* listpack中预留了delete falg，未来会支持从中间删除msg

* stream数据结构涉及3个文件 t_stream.c/listpack.c/rax.c

### 

## 代码学习

### 数据结构体说明

#### raxNode

``` c++
#define RAX_NODE_MAX_SIZE ((1<<29)-1)
typedef struct raxNode {
	/* 表示这个节点是否包含key
	 * 1: 从头部到其父节点的路径完整存储了一个key
	 * 0: 暂不构成key
	 */
    uint32_t iskey:1;    
	/* 是否有存储value值，value值也是存储在data中
	 * 0: 是；1：否
	 */
    uint32_t isnull:1;
	/* 是否有前缀压缩 */
    uint32_t iscompr:1;
	/* 该节点存储的字符个数，
	 * iscompr==1，size表示压缩字符个数
	 * iscompr==0, size表示字符个数
	 */
    uint32_t size:29;     
    /* Data layout is as follows:
     * [header iscompr=0][abc][a-ptr][b-ptr][c-ptr](value-ptr?)
     * [header iscompr=1][xyz][z-ptr](value-ptr?)
     */
    unsigned char data[];
} raxNode;

typedef struct rax {
    raxNode *head;	/* rax树头结点指针 */
    uint64_t numele;/* rax树中元素个数 */
    uint64_t numnodes;/* rax树中node数量 */
} rax
```

#### stream

``` c++
/* Stream item ID: a 128 bit number composed of a milliseconds time and a sequence counter. 
 * XADD时可以手动指定streamID; 也可用*替代，让redis自动生成ID
 */
typedef struct streamID {
    uint64_t ms;        /* Unix time in milliseconds. */
    uint64_t seq;       /* Sequence number. */
} streamID
typedef struct stream {
    rax *rax;               /* The radix tree holding the stream. */
    uint64_t length;        /* Number of elements inside this stream. */
    streamID last_id;       /* Zero if there are yet no items. */
    rax *cgroups;           /* Consumer groups dictionary: name -> streamCG */
} stream

/* 消费组管理结构体. */
typedef struct streamCG {
    streamID last_id;       /* 本消费组目前消费位置游标 */
	 /* pending entries list, 已经发送给本stream group，但没有收到ACK的streams
	  * key: steamID
	  * value: streamNACK 结构体指针 */
    rax *pel; 
	/* key: consumer name
	   name: streamConsumer结构体指针 */	
    rax *consumers;         
} streamCG;

/* 消费者管理结构体  */
typedef struct streamConsumer {
    mstime_t seen_time;         /* Last time this consumer was active. */
    sds name;                   /* 消费者名称，大小写敏感. */
    rax *pel;                   /* 被本消费者挂起streams树：消费后，没有ack的消息；和streamCG->pel对应
								 * key: message ID
								 * value: streamNACK结构体指针 */
} streamConsumer

/* 一个消费组中被挂起的消息 */
typedef struct streamNACK {
    mstime_t delivery_time;     /* Last time this message was delivered. */
    uint64_t delivery_count;    /* Number of times this message was delivered.*/
    streamConsumer *consumer;   /* 本消息最近一次的消费者. */
} streamNACK

/* Stream propagation informations, passed to functions in order to propagate
 * XCLAIM commands to AOF and slaves. */
typedef struct sreamPropInfo {
    robj *keyname;
    robj *groupname;
} streamPropInfo;
```

## 参考资料

[redis rax tree介绍](https://baijiahao.baidu.com/s?id=1631234934342227091&wfr=spider&for=pc)

