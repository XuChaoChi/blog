---
title: 使用vscode(gdb)调试redis
tags:
  - Redis
  - gdb
category: 'Redis学习'
keywords: 'redis,gdb,vscode'
date: 2019-02-14 01:36:05
---


# 使用vscode(gdb)调试redis

想学习redis源码,网上都是推荐gdb,但是终端调试不直观,想着vscode也是gdb调试的就试了下,结果发现可行

环境:

ubuntu 18.04

vscode version: 1.31.1


## 下载与编译redis

下载redis 
    
    $ wget https://github.com/antirez/redis/archive/5.0.2.tar.gz 
    $ tar -zvxf 5.0.2.tar.gz
    $ cd redis-5.0.2
    $ make

<!--more-->

## 下载安装vscode

    $ wget https://az764295.vo.msecnd.net/stable/1b8e8302e405050205e69b59abb3559592bb9e60/code_1.31.1-1549938243_amd64.deb
    $ sudo dpkg -i code_1.31.1-1549938243_amd64.deb

## 调试redis

在菜单栏中找到File>Open Folder(ctrl+k ctrl+o)打开redis的src目录

然后找到Debug>Start Debugging(f5),然后如图选择c++(GDB/LLDB)

![choose gdb](/blog/img/redis_gdb/1.png)

选择之后在左边的.vscode目录下会出现launch.json文件

将 *"program"* 修改为 *"${workspaceFolder}/redis-server"*

![change json](/blog/img/redis_gdb/2.png)

然后打开server.c,找到main函数,并且打上断点,然后继续按f5调试,接下来就能愉快的开始redis学习之旅了

![start debugging](/blog/img/redis_gdb/3.png)