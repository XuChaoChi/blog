---
title: redis源码学习--sds
date: 2019-04-16 02:23:37
tags:
  - Redis
category: 'Redis学习'
keywords: 'redis,sds'
---


# redis源码学习--sds

## 版本与文件信息

版本:[redis5.0.2](https://github.com/antirez/redis/archive/5.0.2.tar.gz )
文件:sds.h sds.c
源码阅读记录地址:[sds阅读记录地址](https://github.com/XuChaoChi/redis_read/blob/master/redis-5.0.2/src/sds.h)
sds(simple dynamic string)简单动态字符串是二进制安全的存储结构,是redis对字符操作的一些简单封装,和传统的c字符不同,sds有个可变长的头部用于存储一个sds串的使用情况,使得其中间可以包含\0,同时封装的一些操作函数(类型装换,扩容,fmt等等操作)

<!--more-->

## sds基本结构

### sds类型

    #define SDS_TYPE_5  0
    #define SDS_TYPE_8  1
    #define SDS_TYPE_16 2
    #define SDS_TYPE_32 3
    #define SDS_TYPE_64 4

TYPE后面的数字用来代表存储的sds最大长度(uint)其中SDS_TYPE_5不存数据

### sds存储的结构体

    typedef char *sds;//sds本质是一个包含头部信息的c字符串
注:sdshdr5的长度存于flag的其他有单独的len

|type|len|alloc|flag|buf|
|-|-|-|-|-|
|sdshdr5|/|/|unsigned char|char*|
|sdshdr8|uint8_t|uint8_t|unsigned char|char*|
|sdshdr16|uint16_t|uint16_t|unsigned char|char*|
|sdshdr32|uint32_t|uint32_t|unsigned char|char*|
|sdshdr64|uint64_t|uint64_t|unsigned char|char*|


## sds的一些宏定义
    
    #define SDS_TYPE_MASK 7//和SDS_TYPE与求类型(类型掩码)
    #define SDS_TYPE_BITS 3//类型占用3位
    #define SDS_HDR_VAR(T,s) struct sdshdr##T *sh = (void*)((s)-(sizeof(struct sdshdr##T)));//获取头部指针
    #define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T))))//获取头部指针
    #define SDS_TYPE_5_LEN(f) ((f)>>SDS_TYPE_BITS)//右移3位求sdshdr5的长度

## 函数

### 静态函数

|函数|作用|
|-|-|
|static inline size_t sdslen(const sds s)|获取sds的长度|
|static inline size_t sdsavail(const sds s)|剩余可用长度|
|static inline void sdssetlen(sds s, size_t newlen)|更新sds长度(len)|

### 外部会调用的函数
|函数|作用|
|-|-|
|sds sdsnewlen(const void *init, size_t initlen)|创建一个新的sds字符串,如果init是NULL或者SDS_NOINIT将创建一个新的sds,sds允许中间有\0 printf只能打印到\0|
|sds sdsnew(const char *init)|将c语言字符串转换成sds,实际用的是sdsnewlen|
|sds sdsempty(void)|创建一个空的sds字符串,实际用的是sdsnewlen|
|sds sdsdup(const sds s)|返回一个拷贝的sds串,实际用的是sdsnewlen|
|void sdsfree(sds s)|释放一个sds字符串,和sdsclear不同,buf会被清除|
|sds sdsgrowzero(sds s, size_t len)|增加sds的长度,增加部分置0,如果增加长度小于原来长度啥都不做|
|sds sdscatlen(sds s, const void *t, size_t len)|添加规定长度的字符串在sds后面,调用后再使用的话要用返回的指针,原来传进去s已经不能用了(sdsMakeRoomFor会重新分配)|
|sds sdscat(sds s, const char *t)|添加c字符串到sds,实际使用sdscatlen()|
|sds sdscatsds(sds s, const sds t)|添加sds串到目标sds,实际使用sdscatlen()|
|sds sdscpylen(sds s, const char *t, size_t len)|破坏性修改s(修改后使用返回值),添加t开始len长度的字符串,中间可以包含\0|
|sds sdscpy(sds s, const char *t)|和sdscpylen差不多,不过结尾需要\0(strlen需要)|
|sds sdscatvprintf(sds s, const char *fmt, va_list ap)|fmt到缓冲区,当长度不够时每次失败的时候扩大2倍|
|sds sdscatprintf(sds s, const char *fmt, ...)|sds format调用的sdscatvprintf|
|sds sdscatfmt(sds s, char const *fmt, ...);|类似sprintf()但是比sprintf效率高,和printf类型的使用方法不同,%s是c字符串,%S是sds,%i的有符int,%I是int64_t,%u是uint32_t,%U是uint64_t,%%就是%|
|sds sdstrim(sds s, const char *cset);|从左右s的左右2断进行修剪,去除包含cset中任意字符,比如"AA...AA.a.aa.aHelloWorld     :::"的字符串从左右去除"Aa. :"中的字符,结果为HelloWorld|
|void sdsrange(sds s, ssize_t start, ssize_t end);||
|void sdsupdatelen(sds s);|更新sds的长度,搜索了下貌似没有地方用到了,用strlen的长度更新sds的长度|
|void sdsclear(sds s);|清空sds字符串,不过不释放内存只是单纯的改了长度和在字符串首部添加\0,提升效率这样就不用下次使用的时候重复分配|
|int sdscmp(const sds s1, const sds s2);|对比2个sds的内容,按照小的sds的长度|
|sds *sdssplitlen(const char *s, ssize_t len, const char *sep, int seplen, int *count);|使用关键词sep分割目标位s的字符串,返回分割后的sds数组的指针,并且count为分割的数量|
|void sdsfreesplitres(sds *tokens, int count);|释放sdssplitlen生成的结果,如果tokens是空则什么都不做|
|void sdstolower(sds s);|转换sds成小写|
|void sdstoupper(sds s);|转换sds成大写|
|sds sdsfromlonglong(long long value);|longlong转string,比sdscatprintf快很多|
|sds sdscatrepr(sds s, const char *p, size_t len);|在原基础上截取索start到end的字符串,也可以倒序-1是最后一个-2是最后第二个|
|sds *sdssplitargs(const char *line, int *argc);|判断是否是16进制,在sdssplitargs中使用|
|sds sdsmapchars(sds s, const char *from, const char *to, size_t setlen);|用来替换sds指定中的内容,比如sdsmapchars(mystring, "ho", "01", 2),如果mysstring是"hello"的话"h"和"o"分别将被替换成"0","1",结果就是"0ell1"|
|sds sdsjoin(char **argv, int argc, char *sep);|遍历添加c字符串数组到sds|
|sds sdsjoinsds(sds *argv, int argc, const char *sep, size_t seplen);|遍历添加sds数组字符串到sds|

### 一般内部sds内部调用函数
|函数|作用|
|-|-|
|sds sdsMakeRoomFor(sds s, size_t addlen);|扩容用的,在末尾添加未使用的空间,不改变使用长度,调用sdslen()还是返回原来的长度|
|void sdsIncrLen(sds s, ssize_t incr);|增长或减少sds的l实际使用长度(len)|
|sds sdsRemoveFreeSpace(sds s);|重新分配sds使得末尾没有空闲空间,已经使用的空间仍然没有改变,调用之后原来的sds指针将无效,所有的引用要换成新的返回的sds指针|
|size_t sdsAllocSize(sds s);|返回所有的分配空间包含头的长度.分配的长度(包含使用和未使用),\0的长度|
|void *sdsAllocPtr(sds s);|返回sds实际分配的指针(减掉头部分)|

### 封装zmalloc的函数
|函数|作用|
|-|-|
|void *sds_malloc(size_t size);|分配内存|
|void *sds_realloc(void *ptr, size_t size);|重新分配内存|
|void sds_free(void *ptr);|释放内存|

## 参考
《redis的设计和实现》第二章