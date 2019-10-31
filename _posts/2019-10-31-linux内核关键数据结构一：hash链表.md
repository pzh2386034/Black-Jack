---
title: linux内核关键数据结构一：hash链表
date: 2019-10-31
categories:
- linux-algorithm
tags:
- hash, list,hlist
---

## 前沿

>linux内核代码对效率及空间的要求是极高的，这对于我们想写出高性能，高质量的代码有巨大的参考价值，它就像一座金矿，等着被人慢慢挖掘；接下来我想用几篇来学习下内核中的常见数据结构，及用个小栗子使用它；

### 本篇主要内容

学习hlist的相关方法及使用场景

* hlist定义在内核hlist.h文件中的数据结构，主要用在解决hash冲突时使用chaining方法时使用

* htable是hash数组，每个数组元素是一个hlist_head结构体；当多个节点的hash值相同时，直接添加在对应的结点链表中即可

## 代码分析

### 结构体及重要方法分析

#### hlist_head, hlist_node

``` c++
/* 链表头 */
struct hlist_head{
  struct hlist_node * first;
  /* 链表头使用前必须要初始化 */
  hlist_head():first(NULL){};
};
/* 链表节点，具体的数据结构体包含本结构体即可达到链接效果 */
struct hlist_node{
  struct hlist_node* next;
/* 
 * 二级指针，指向前一个节点的next指针地址，why？
 * 显然，如果hlist_node采用传统的next,prev指针，对于第一个节点和后面其他节点的处理会不一致 
 * 由于hlist_head->first的结点类型和hlist_node->next结点类型相同，这样就解决了通用性！
 */
  struct hlist_node ** pprev;
};
/* 附带的一些数据初始化方法，无论是head、node，使用前一定要初始化 */
#define HLIST_INIT_HEAD(ptr) ((ptr)->first = NULL;)
#define HILIST_HEAD(name) hHead name = {.first = NULL};
static inline void HLIST_INIT_NODE(struct hlist_node * h)
{
  h->next = NULL;
  h->pprev = NULL;
}
```


### 增、删、查关键方法

>遍历及相关方法见[附录](#附录)

#### hlist_add_head

>在某个hash bulk中添加节点的相关函数

``` c++
/* hash桶头结点之后，插入第一个普通结点 */
static inline void hlist_add_head(struct hlist_node* n, struct hlist_head *h)
{
/* 无需死记操作步骤，从简单到复杂分3步走
 * 1. 初始化新节点的指针
 * 2. 修复桶头节点first指针，指向新节点
 * 3. 修复后面节点pprev指针，指向新节点->next指针
 */
  hlist_node* first = h->first;
  n->next = first;
  n->pprev = &first;
  
  h->first = n;
  
  if (first)
    {
	/* hlist_head必须要初始化，否则第一次添加数据在次会coredump */
      h->first->pprev = &n->next;
    }
}
/* 在before节点前，添加新节点 */
static inline void hlist_add_before(struct hlist_node *n, struct hlist_node *before)
{
/* 还是类似3步走即可 */
  hlist_node *after_next = *before->pprev;
  n->next = before;
  n->pprev = before->pprev;

  before->pprev = &n->next;
  
  after_next = n;
}
/* 在指定节点后，添加新节点 */
static inline void hlist_add_after(struct hlist_node* n, struct hlist_node* after)
{
  n->next = after->next;
  n->pprev = &after->next;

  if (after->next)
    {
      after->next->pprev = &n->next;
    }
  after->next = n;
}
```

#### hlist_del_init

>删除指定节点，并将被删节点数据初始化

``` c++
static inline void _hlist_del(struct hlist_node * n)
{
/* 无需死记操作步骤，分2步走
 * 1. 修复前面节点next指针
 * 2. 修复后面节点pprev指针
 */
  hlist_node* pre_node_next = *n->pprev;
  hlist_node* next_node = n->next;
  pre_node_next = n->next;

  if (next_node!=NULL){
    next_node->pprev = &pre_node_next;
  }

}
static inline void hlist_del_init(struct hlist_node *n)
{
  _hlist_del(n);
  n->next = NULL;
  n->pprev = NULL;
}
```

#### main

>测试程序，意图很明显，配合hlist.h使用即可;c++,c混合，有点混乱

``` c++
#include "hlistdemo.h"
#include "stdlib.h"
#include "unistd.h"
#include <iostream>
#include "stdio.h"

using namespace::std;

struct hdata_node{
  unsigned int data;
  struct hlist_node list;
  hdata_node(int _data):data(_data){HLIST_INIT_NODE(&list);};
  hdata_node(){HLIST_INIT_NODE(&list);};
};
static inline void hlist_add_head(struct hlist_node* n, struct hlist_head* h);

static inline void hlist_del_init(struct hlist_node *n);

static inline int hlist_empty(const struct hlist_head* h);
void print_hlist(hlist_head* h)
{
  struct hdata_node* pos;
  hlist_for_each_entry(pos, h, list)
    {
      cout<<pos->data<<"  ";
    }
  cout<<endl;
}
void insert_hdata_node(unsigned int  d, struct hlist_head*h)
{
  struct hdata_node * hdata = new struct hdata_node();
  hdata->data = d;
  hlist_add_head(&hdata->list, h);
}
bool delete_hdata_node(unsigned int d, struct hlist_head* h)
{
  struct hlist_node *tmp;
  struct hdata_node* hdata;
  hlist_for_each_entry_safe(hdata, tmp, h, list){
    if(hdata->data == d)
      {
        printf("find val: %u, begin to delete\n", d);
        hlist_del_init(&hdata->list);
        delete hdata;
        hdata = NULL;
      }
    else if(hdata->list.next == NULL)
      {
        break;
      }
  }
  print_hlist(h);
  return 0;
}
int main()
{
  int quit = 1;
  struct hdata_node* hdata = NULL;
  struct hlist_head htable[5] , hhead ;
  struct hlist_node* hlist = NULL, *n = NULL;
  int data = 0;


  while(quit!=0)
    {

      cout<<endl;
      cout<<endl;
      cout<<endl;
      printf("*********************\n\n"
          "input options:\n"
          "1: insert\n"           //插入
          "2: delete\n"           //删除
          "0: quit\n"
          "\n*********************\n");  
      int in = 0;
      cin>>in;
      switch(in)
        {
        case 0:
          quit = 0;
          cout<<"quit test."<<endl;
          break;
        case 1:
          cout<<"please input insert data."<<endl;
          cin>>data;
          insert_hdata_node(data, &htable[data%5]);
          cout<<"print all data in this bulk: "<<endl;
          print_hlist(&htable[data%5]);
          break;
        case 2:
          cout<<"please input delete data."<<endl;
          cin>>data;
          if (htable[data%5].first != NULL)
            delete_hdata_node(data, &htable[data%5]);
          else
            cout<<"empty hash table,delete failed."<<endl;
          break;
        }
    }
  for(int i =0; i<5; i++)
    {
      cout<<"delete hash table "<<i<<endl;
      hlist_for_each_entry_safe(hdata, n, &htable[i], list)
        {
          cout<<"delete node: "<<hdata->data<<"  ";
          hlist_del_init(&hdata->list);
          delete hdata;
          hdata = NULL;
        }
      cout<<endl;
    }
}
```


## 附录

### 获取ptr指针所在hnode的结构体指针

``` c++

#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)

#define container_of(ptr, type, member) ({\
      const typeof( ((type *)0)->member ) * __mptr = (ptr);  \
      (type*)((char*)__mptr - offsetof(type, member) );})

/* 
 * 1. ptr: 结构体成员变量指针
 * 2. type: 结构体
 * 3. member: 结构体中ptr类型的变量名
 * 返回：ptr所在结构体的指针首地址
 */
#define hlist_entry(ptr, type, member) container_of(ptr,type,member)
struct hlist_node * hlist1;
struct hlist_node * next = hlist1->next;
struct hlist_node * hlist2 = hlist_entry(next, struct hlist_node, next);
/* 则hlist1==hlist2 */

#define hlist_entry_safe(ptr, type, member) \
  ({typeof(ptr) __ptr = (ptr);                      \
    __ptr ? hlist_entry(__ptr, type, member): NULL;\
  })
```

### 遍历一个hash bulk

``` c++
/* 
 * 1. head: hash桶头指针hlist_head*
 * 2. pos: 节点游标，用于for循环使用
 */
#define hlist_for_each(pos, head)\
  for(pos = (head)->first; pos; pos = pos->next)
/* 安全版，用于for循环体中有删除节点动作 */
#define hlist_for_each_safe(pos, head, tmp)\
  for(pos = (head)->first; pos && ({tmp = pos->next; 1; }); pos = tmp)
```

### 遍历hash bulk，返回数据结构体

``` c++
/* 
 * 1. pos: 节点游标，用于for循环使用
 * 2. head: hash桶头指针hlist_head*
 * 3. member: hlist_node在数据结构体中名称
 */
#define hlist_for_each_entry(pos, head, member) \
  for(pos = hlist_entry_safe((head)->first, typeof(*(pos)), member);  \
          pos;\
      pos = hlist_entry_safe((pos)->member.next, typeof(*(pos)), member))

struct hdata_node{
	unsigned int data;
	struct hlist_node list;
}
struct hlist_node* hlist;
struct hlist_head* hhead;
struct hdata_node* hdata;
hlist_for_each_entry(hdata, hhead, list)
{
....
}
/* 上面的循环方法等价于下面这个 */
hlist_for_each(hlist, hhead)
{
	hdata = container_of(hlist, struct hdata_node, list)
	...
}

/* 安全版，用于for循环体中有删除节点动作 */
#define hlist_for_each_entry_safe(pos,n, head, member)\
      for(pos = hlist_entry_safe((head)->first, typeof(*pos), member);\
          pos && ({n = pos->member.next; 1; });\
          pos = hlist_entry_safe(n, typeof(*pos), member))

```

## 参考资料

[hlist实例](https://blog.csdn.net/fuyuande/article/details/81039072)

[hlist解析](https://blog.csdn.net/hs794502825/article/details/24597773)
