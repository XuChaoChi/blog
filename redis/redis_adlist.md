---
title: redis源码学习--adlist
date: 2019-04-20 19:01:51
tags:
  - Redis
category: 'Redis学习'
keywords: 'redis,adlist'
---

# redis源码学习--adlist
## 版本与文件信息
版本:[redis5.0.2](https://github.com/antirez/redis/archive/5.0.2.tar.gz )
文件:adlist.h adlist.c
源码阅读记录地址:[sds阅读记录地址](https://github.com/XuChaoChi/redis_read/blob/master/redis-5.0.2/src/adlist.h)
list是redis底层实现的一个双向链表

<!--more-->

## 结构

### 节点结构

    typedef struct listNode {
        struct listNode *prev;//前驱
        struct listNode *next;//后继
        void *value;//值
    } listNode;

### 链表管理结构

    typedef struct list {
        listNode *head;//头指针
        listNode *tail;//尾指针
        //针对不同类型值的一下函数指针
        void *(*dup)(void *ptr);//复制函数指针
        void (*free)(void *ptr);//释放函数指针
        int (*match)(void *ptr, void *key);//对比函数指针
        unsigned long len;//链表长度
    } list;

为了适应不同数据的复制,释放,和对比list提供了对应的函数指针

### 迭代器结构

    typedef struct listIter {
        listNode *next;
        int direction;//迭代器访问方向
    } listIter;

    //方向的枚举类型
    #define AL_START_HEAD 0//头开始
    #define AL_START_TAIL 1//尾开始

迭代器的实现方便了list的顺序访问,可以通过direction实现向前和向后遍历,遍历样例

    iter = listGetIterator(list,<direction>);
    while ((node = listNext(iter)) != NULL) {
        doSomethingWith(listNodeValue(node));
    }

## 操作list结构的函数
    //redis操作list的函数因为相对简介,所以直接用宏来实现
    #define listLength(l) ((l)->len) //获取长度
    #define listFirst(l) ((l)->head) //获取头
    #define listLast(l) ((l)->tail) //获取尾
    #define listPrevNode(n) ((n)->prev) //获取前置节点
    #define listNextNode(n) ((n)->next) //获取后置节点
    #define listNodeValue(n) ((n)->value) //获取值

    #define listSetDupMethod(l,m) ((l)->dup = (m))  //设置复制函数指针
    #define listSetFreeMethod(l,m) ((l)->free = (m)) //设置释放函数指针
    #define listSetMatchMethod(l,m) ((l)->match = (m)) //设置对比函数只恨

    #define listGetDupMethod(l) ((l)->dup) //获取复制函数指针
    #define listGetFree(l) ((l)->free)  //获取释放函数指针
    #define listGetMatchMethod(l) ((l)->match) //获取对比函数指针

## 其他函数

|函数|作用|
|-|-|
|list *listCreate(void); |创建一个空的链表,不包含数据节点|
|void listRelease(list *list);|释放list的数据但是不删除这个list本身|
|void listEmpty(list *list);|释放所有的list,包括头|
|list *listAddNodeHead(list *list, void *value);|在头部添加一个新的list节点,失败返回空,list不变,成功返回成功后的list|
|list *listAddNodeTail(list *list, void *value);|添加一个新的节点到尾部,失败返回空,list不变,成功返回成功后的list|
|list *listInsertNode(list *list, listNode *old_node, void *value, int after);|插入一个节点|
|void listDelNode(list *list, listNode *node);|删除一个规定的节点在规定的链表中,同时也会释放节点的私有值,给迭代器|
|listIter *listGetIterator(list *list, int direction);|获取list的迭代器,通过迭代器listNext()可以实现顺序访问|
|listNode *listNext(listIter *iter);|迭代器访问下一个元素,可以使用listDelNode删除当前节点但是不能删除其他的节点,防止访问混乱|
|void listReleaseIterator(listIter *iter);|释放迭代器|
|list *listDup(list *orig);|复制整个链表,失败返回NULL,不会修改原来的链表|
|listNode *listSearchKey(list *list, void *key);|查找一个第一个(从头开始)符合match规则的节点|
|listNode *listIndex(list *list, long index);|返回指定索引的节点,正反序都可以,不在范围内返回空|
|void listRewind(list *list, listIter *li);|重置迭代器以list的头方向|
|void listRewindTail(list *list, listIter *li);|重置迭代器以list的尾方向|
|void listRotate(list *list);|反转链表,其实就是头尾互换下,O(1)|
|void listJoin(list *l, list *o);|把o的节点转移到l上来,其实就是把l的尾节点指向o的头节点|
