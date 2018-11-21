---
title: Redis导读(README翻译)
abbrlink: c1b13d65
date: 2018-07-12 01:17:00
tags: Redis
category: Redis学习
thumbnail: 'https://blog.cxc233.com/blog/img/20180712/1.jpg'
keyworks: redis,导读,readme
---

![logo](/blog/img/20180712/1.jpg)

这个博客就是一个redis的快速导读。你可以从[redis.io](https://redis.io).找到更多的细节。 
这篇博客翻译于本人开始阅读Redis之前所以难免有错误和不到位的地方欢迎指正
## 什么是Redis? ##
--------------

Redis经常被称为**数据结构**服务器。 这意味着Redis通过一组命令提供对可变数据结构的访问, 使用**C/S**架构通过发送简单的TCP协议来交互。因此，不同的进程可以以共享方式查询和修改相同的数据结构。

Redis中实现的数据结构具有一些特殊属性:

* Redis可以将数据存放到磁盘中, 即使他们总是在内存中被获取和修改。 这意味着Redis不仅速度快, 而且数据不容易丢失。
* 使用数据结构是为了强调内存的效率, 所以在redis中的数据结构模型和其他的高级语言相比可能会使用更少的内存。
* Redis提供了很多传统数据库使用的特性，比如：迁移, 可调控的数据持久化, 集群, 高性能.

另一个很好的理解是将Redis视为**memcached**的一个更复杂的版本，其中操作不仅仅是SET和GET，而且还可以处理复杂数据类型的操作，如链表，集合，有序数据结构等等。

如果你想知道更多关于Redis的信息, 下面提供了很好的开始点:

* 介绍Redis的数据类型. [http://redis.io/topics/data-types-intro](http://redis.io/topics/data-types-intro)
* 直接在浏览器中尝试Redis操作. [http://try.redis.io](http://try.redis.io)
* Redis所有的操作命令. [http://redis.io/commands](http://try.redis.io)
* 查看更多的官方文档. [http://redis.io/documentation](http://redis.io/documentation)
<!--more-->
## 构建 Redis ##
--------------
在Redis中实现的数据结构具有一些特殊属性
Redis能够在Linux, OSX, OpenBSD, NetBSD, FreeBSD环境下编译
Redis提供了大端和小端的架构, 并且都支持32位和64位。

Redis也可以被编译在Solaris系列的操作系统 (例如SmartOS) 但是不能保证很好的像Linux, OSX, 和 BSD等等操作系统一样良好运行


构建很简单:

    % make

选择构建32位的Redis:

    % make 32bit

在构建之前可以先试下make命令是否可用:

    % make test
## 修复依赖项或缓存构建项的问题 ##

---------
Redis 依赖了在 `deps` 目录下的一些库。
`make`不会自动重新构建就算是依赖项的源文件被改变

当你使用`git pull`或者代码内的依赖树以任何方式改变的时候，请务必使用以下内容
命令，以便真正的清空工程重新构建：


    make distclean

这会清理: jemalloc, lua, hiredis, linenoise这些依赖项。

此外，如果您强制某些构建选项，如32位,没有C编译器优化（用于调试目的）和其他类似的构建时间选项，
这些选项会无限期缓存，直到您发出`make distclean`命令。

## 处理在编译32位时候产生的问题 ##
---------

如果你在编译了32位Redis后需要继续编译64位或者相反，你需要在Redis发行版的更目录下运行`make distclean`


在编译32位的Redis出错的时候, 可以尝试以下步骤:

* 安装 libc6-dev-i386 (也尝试下 g++-multilib).
* 用以下命令代替 `make 32bit`:
  `make CFLAGS="-m32 -march=native" LDFLAGS="-m32"`

## 分配器 ##
---------
在构建Redis时通过设置`MALLOC`环境变量选择非默认内存分配器。
Redis在默认情况下使用c函数库来编译和链接的， 除了在linux下使jemalloc。 因为使用jemalloc没有c函数库那么多的问题在linux下
。
强制编译libc编译,使用:

    % make MALLOC=libc

在Mac系统上强制使用jemalloc:

    % make MALLOC=jemalloc

## 详细构件 ##
-------------
默认情况下，Redis将使用用户友好的彩色输出进行构建。
如果要查看更详细的输出，请使用以下内容:

    % make V=1

## 运行Redis ##
-------------

使用默认配置运行Redis：

    % cd src
    % ./redis-server

如果你需要使用redis.conf来运行redis，你必须要添加额外点的参数（配置文件的的路径）：

    % cd src
    % ./redis-server /path/to/redis.conf

也可以直接传递参数来改变Redis的配置，使用命令行的例子：

    % ./redis-server --port 9999 --slaveof 127.0.0.1 6379
    % ./redis-server /etc/redis/6379.conf --loglevel debug

所有在redis.conf的配置都支持用命令行来更改，只要和参数名字一样就可以了

## 使用Redis ##
------------------

你可以使用redis-cli来启动Redis客户端。开始一个redis-server的实例，然后再其他终端尝试下面的命令:

    % cd src
    % ./redis-cli
    redis> ping
    PONG
    redis> set foo bar
    OK
    redis> get foo
    "bar"
    redis> incr mycounter
    (integer) 1
    redis> incr mycounter
    (integer) 2
    redis>

你能在[http://redis.io/commands](http://redis.io/commands)找到所有可以用的命令。


## 安装Redis ##
-----------------

安装 Redis 到默认目录 /usr/local/bin 使用:

    % make install

也可以使用 `make PREFIX=/some/other/directory install`到其他目录

`% make install`只能把可执行文件安装到你的系统，但是不会有配置信息

初始化脚本和配置文件在合适的地方。如果你很少用Redis就不是太需要，但是如果想在安装的时候正确配置，Redis为Ubuntu和Debian提供了脚本：

    % cd utils
    % ./install_server.sh

这个脚本会问询问一系列的设置问题，然后作为后台守护进程在系统重启的时候再次启动。

你可以使用叫做`/etc/init.d/redis_<portnumber>`的脚本停止并启动Redis
, 比如 `/etc/init.d/redis_6379`.

## 代码贡献 ##
-----------------

提示: 以任何形式对Redis工程的代码贡献, 包括发送通过Github的提交请求, 电子邮件和公共讨论组, 并且同意根据条款发布您的代码您可以在Redis中包含的[COPYING][1]文件中找到的BSD许可证来源。

请查看[COPYING][2]文件的来源方式查看更多信息。

[1]: https://github.com/antirez/redis/blob/unstable/COPYING
[2]: https://github.com/antirez/redis/blob/unstable/CONTRIBUTING

## Redis的内部构成 ##
===

如果你正在读README你可能在Redis Github的第一页
或者是解压Redis工程后. 再这两种情况下你离源码基本只有一步之遥了, 所以我们在这里解释Redis源代码布局, 首先我们要知道每个文件的内容是什么, Redis服务器内部最重要的功能和结构等等。
我们将大致而不是深入的讲解，因为库文件很大，并且会不断变化，但是作为开始是很好的起点。同时大多数的代码都有评论和简单的跟随。

## 源文件布局构成 ##
---

Redis的根目录就是包含这个README文件的目录，真正的Makefile文件和一个Redis和Sentinel的配置例子在`src`目录。
。你可以找一些shell脚本来执行Redis, Redis集群和Redis Sentinel单元测试, 这些实现在`tests`目录.

在根目录中重要的目录:

* `src`: 包含c语言写的 Redis 的实现。
* `tests`: 单元测试, Tcl(一种脚本语言)实现。
* `deps`: Redis的依赖库。 所有Redis编译需要的库都在这里面; 
您的系统只需要提供`libc`，POSIX兼容接口和C编译器。 值得注意的是`deps`包含了`jemalloc`的副本, Linux下的内存分配器。 (下面的内容没找到。。)`deps`也包含了一下Redis项目开始的事情, 同时`anitrez/redis`不是主要的存储库 。 这个规则也有例外 `deps/geohash-int` 是一个Redis使用的低级地理编码库: 他起源于一个不同的项目, 但是存在比较大的分支，所以作为一个Redis中独立的实体来开发。

同时也有一些其他的目录但是并不是我们的主要目标。
Redis的主要实现在`src`中,所以我们要把主要精力投入到当中去。
探索`src`中的每一个文件时，是随着顺序的进行而逻辑复杂性也会增加。

提示: 最新的Redis会重构相当一部分内容. 函数名和文件名都有可能改变，所以你可以找到这些分支的映射在`unstable`目录中（在github上找）改变的最新文档。比如在Redis3.0中的
`server.c`和`server.h` 被命名为了 `redis.c` 和 `redis.h`。不过无论如何数据结构的名字还是一样的。
记住所有的开发和提交都应该在`unstable` 的分支下去找。

## server.h ##
---

理解程序如何工作的最简单方法是理解它使用的数据结构。
所以我们将从Redis主函数的头文件 `server.h`开始。

所有的配置和共享状态的结构体都是定义成叫做`server`的全局结构体，`struct redisServer`类型。
一些在这个结构体中的重要字段:

* `server.db` 是一个Redis用来存储数据的数据库数组.
* `server.commands` 是一个命令的表.
* `server.clients` 是连接到服务器的客户端的链接列表。
* `server.master` 一个特殊的客户端, 主从结构中的主机。

另一个重要叫做 `redisClient`的结构体数据时关于客户端的定义。这个结构体有很多字段，下面展示了主要部分：

    struct client {
        int fd;
        sds querybuf;
        int argc;
        robj **argv;
        redisDb *db;
        int flags;
        list *reply;
        char buf[PROTO_REPLY_CHUNK_BYTES];
        ... many other fields ...
    }

这个结构体用于 *已经连接的客户端*:

* `fd` 字段是客户端socket的标识符。
* `argc` 和 `argv` 是用来执行客户端的命令, Redis提供了功能来读取命名行参数。
* `querybuf` 累计客户端的请求，由Redis服务端根据协议解析分发到客户端来执行。
* `reply` 和 `buf` 分别是动态和静态缓冲区，用于累积服务器发送给客户端的回复。 一旦文件描述符可写，这些缓冲区就会通过套接字发送给客户端。


正如在上面的客户端结构中所看到的，命令中的参数
被描述为`robj`结构。 以下是完整的`robj`结构体：

    typedef struct redisObject {
        unsigned type:4;
        unsigned encoding:4;
        unsigned lru:LRU_BITS; /* lru time (relative to server.lruclock) */
        int refcount;
        void *ptr;
    } robj;

基本上这个结构可以代表所有基本的Redis数据类型
字符串，列表，集合，排序集等。有趣的事是他有一个`type`字段， 所以他可能知道对象的类型, 以及一个 `refcount`字段, 所以相同的对象可以被引用，避免多次分配造成的内存浪费。 最后 `ptr`字段
指向真正的被引用对象, 不过内容可能会不同, 取决于使用 `encoding` 字段的编码.

Redis对象广泛用于Redis内部, 可以避免许多地方间接访问的内存开销，最近在很多地方我们只使用未包装在Redis对象中的普通动态字符串。

## server.c ##
---

Redis 服务端的入口`main()`函数在这里面定义。接下来是在Redis服务开始之前非常重要的步骤 。

* `initServerConfig()` 用默认值初始化`server`结构体。
* `initServer()` 给需要操作的数据结构分配内存, 启动socket监听和其他的一些事。
* `aeMain()` 开始新连接的事件循环。


事件循环定期调用两个特殊函数:

1. `serverCron()` 被定期调用 (根据 `server.hz` 的频率), 必须不停的执行任务，比如检测客户端的超时。
2. `beforeSleep()` 每一次事件循环都会触发，Redis会处理一些请求，并回到事件循环中去。

在`server.c` 你可以找到Redis server处理其他事情的重要函数:

* `call()` 在客户端中调用给定的命令。
* `activeExpireCycle()` 通过`EXPIRE`命令处理超过生存时间的键失效。
* `freeMemoryIfNeeded()`新的读写命令需要被执行但根据`maxmemory`指令，Redis内存不足。
* 全局变量 `redisCommandTable` 定义了Redis的所有命令，指定命令的名称, 命令的实现, 的需要的参数个数, 和每个命令的其他属性。

## networking.c ##
---

这个文件定义了所有客户端的的 I/O(输入/输出)函数, 主从机
(在Redis中特殊的客户端):

* `createClient()` 给新的客户端分配内存和初始化。
* `addReply*()` 一系列实现的命令行函数用来添加数据结构到客户端， 他们会把命令的执行结果发送到客户端。
* `writeToClient()` 被*可写事件处理*`sendReplyToClient()`调用将输出缓冲区中待处理的数据传输到客户端 。
* `readQueryFromClient()` 是 *可读事件处理* 并将从客户端读取的数据累积到查询缓冲区中。
* `processInputBuffer()` 是根据Redis协议解析客户端查询缓冲区的入口点。 当命令准备好被处理的时候, 他讲调用在 `server.c` 定义的`processCommand()`  来进行实际命令的执行.
* `freeClient()` 释放, 断开和移除一个客户端。

## aof.c and rdb.c ##
---

从名字上可以看出他是Redis持久化的实现RDB 和 AOF的地方。 Redis使用持久化模型基于调用 `fork()`
，在主线程创建具有相同（共享）内存的线程
，次线程存储内存内容要磁盘上。 这是`rdb.c`用来在磁盘上创建快照 和`aof.c` 执行 AOF 重写当添加文件太大的时候使用的。在 `aof.c`实现的一些额外实现的函数是为了允许新命令附加到AOF文件，客户端来执行他们。

 `call()` 函数在`server.c`中定义的一个主管将命令写入 AOF.

## db.c ##
---


某些Redis命令对特定数据类型进行操作, 其他是通用的。例如通用命令 `DEL` 和 `EXPIRE`。 这些命令是在Key上操作的不，而不是他们特殊的值。 所有的通用命令都被定义在 `db.c`。


此外，`db.c`实现API，在Redis数据集上来执行某些没有直接访问内部数据结构操作。
在`db.c`中最重要的是实现了很多命令函数：

* `lookupKeyRead()` 和 `lookupKeyWrite()` 用来或者去指定件的指针的值如果中不存在返回`NULL`。
* `dbAdd()` 是更高级的`setKey()` 在Redis中创建一个新的键值.
* `dbDelete()` 删除一个键和对应的值。
* `emptyDb()` 删除一个整个的数据库或者整个数据库的定义。

文件的其他部分实现了客户端的通用命令接口。

## object.c ##
---

在Redis的`robj`结构对象中已经描述过。
在`object.c`中 有所有与Redis对象一起运行的函数基函数，如分配新对象的函数，处理引用计数等等，在文件中的函数:

* `incrRefcount()` 和 `decrRefCount()` 用力增加和减少引用, 当减少到0时对象被释放。
* `createObject()` 分配一个新的对象. 还有专门的函数来分配具有特定内容的字符串对象, 像 `createStringObjectFromLongLong()` 和相似的函数。

这个文件也实现 `OBJECT` 的命令。

## replication.c ##
---

这是在Redis里面最复杂的文件之一, 建议在熟悉其余代之后再去阅读它。
在此文件中，实现了Redis的主从角色。

在这个文件里面最重要的函数之一 `replicationFeedSlaves()` 他可以向客户端写命令来代表从机实例来连接我们的主机，这样从机就可以获得客户端的写操作:通过这样他们的数据集将与主机数据集保持同步。这个文件也实现了 `SYNC` 和 `PSYNC` 命令，用于执行主机和从机间的第一次同步或在断开连接后的继续复制。

## 其他 C 文件 ##
---

* `t_hash.c`, `t_list.c`, `t_set.c`, `t_string.c` and `t_zset.c` 中包含了Redis的数据类型的实现。 它们实现了一个API来访问给定的数据类型, 和实现客户端对于这些类型的命令。
* `ae.c` 实现了Redis的事件循环, 它是一个独立的库，比较易于阅读和理解。
* `sds.c` 是Redis的字符串库, [http://github.com/antirez/sds](http://github.com/antirez/sds)有更多的信息。
* `anet.c` 与内核公开的原始接口相比，它是一种以更简单的方式使用POSIX网络的库。
* `dict.c` 逐渐实现一个非阻塞的Hash表。
* `scripting.c` 实现Lua脚本. 它完全独立于Redis实现的一部分，并且很容易理解如果熟悉Lua API的话。
* `cluster.c` 实现Redis集群。可能在熟悉Redis代码库的其余部分之后，才能比较好阅读与理解. 如果要 `cluster.c` 确保已经读过 [Redis Cluster specification][3].

[3]: http://redis.io/topics/cluster-spec

## 解析Redis命令 ##
---

所有的Redis命令都以下面的方式定义:

    void foobarCommand(client *c) {
        printf("%s",c->argv[1]->ptr); /* Do something with the argument. */
        addReply(c,shared.ok); /* Reply something to the client. */
    }

命令在 `server.c` 的命令中被引用:

    {"foobar",foobarCommand,2,"rtF",0,NULL,0,0,0,0,0},

在上面的例子 `2` 命令有的参数的个数,`"rtF"`是命令的标志, 作为 `server.c`命令表中的描述。

命令以某些方式运行后他会返回打客户端,
通常是使用 `addReply()`或者是在`networking.c`定义的一些相似的函数。

Redis源代码中有大量的命令实现，它可以作为实际命令使用的参考。来写一些测试命令可以很好地练习熟悉代码库.

也有很多不是经常使用的文件没有提到。 我们希望能在第一步上对你有所帮助。
最终，您将在Redis代码库中找到自己的方式 :-)

Enjoy!


