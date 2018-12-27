---
title: LINUX 端口监听Too many open files
tags: [linux]
thumbnail: 
category: linux
keywords: linux,ulimit
---

## LINUX 端口监听Too many open files

### 查看命令　
使用**ulimit -n**命令查看最多的打开的文件描述符个数，如果大于这个数就会出现

_[warn] Error from accept() call: Too many open files_

### 修改方法

#### 临时间方法
使用 ulimit -n 10240改变当前用户的文件描述符个数，重启后无效

<!--more-->

#### 长久方法
vim /etc/security/limits.conf 
在样例后面添加*是所有用户

\* &emsp;&emsp;soft&emsp;nofile &emsp;&emsp;65536

\* &emsp;&emsp;hard&emsp;nofile&emsp;&emsp;65536

如果这样不行的话可以试试指定用户名（我就不行）

root &emsp;&emsp;soft&emsp;nofile &emsp;&emsp;65536

root &emsp;&emsp;hard&emsp;nofile&emsp;&emsp;65536

### 修改其他
通过 ulimit --help可以看到能修改的还有以下选项

    -S	使用软 (`soft') 资源限制
    -H	使用硬 (`hard') 资源限制
    -a	所有当前限制都被报告
    -b	套接字缓存尺寸
    -c	创建的核文件的最大尺寸
    -d	一个进程的数据区的最大尺寸
    -e	最高的调度优先级 (`nice')
    -f	有 shell 及其子进程可以写的最大文件尺寸
    -i	最多的可以挂起的信号数
    -k	分配给此进程的最大 kqueue 数量
    -l	一个进程可以锁定的最大内存尺寸
    -m	最大的内存进驻尺寸
    -n	最多的打开的文件描述符个数
    -p	管道缓冲区尺寸
    -q	POSIX 信息队列的最大字节数
    -r	实时调度的最大优先级
    -s	最大栈尺寸
    -t	最大的CPU时间，以秒为单位
    -u	最大用户进程数
    -v	虚拟内存尺寸
    -x	最大的文件锁数量
    -P	最大伪终端数量
    -T	最大线程数量

