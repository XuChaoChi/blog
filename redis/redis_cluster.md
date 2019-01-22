---
title: redis集群部署
tags:
  - Redis
category: 'Redis学习'
keywords: 'redis,cluster'
date: 2019-01-23 01:36:05
---

# redis集群部署

## 下载安装redis

### 安装环境

为了模拟集群环境,虚拟机安装了6个(也可以在一台电脑上安装6个) centos 7.4 x64

虚拟机的ip分别是

    192.168.1.150
    192.168.1.151
    192.168.1.152
    192.168.1.153
    192.168.1.154
    192.168.1.155

### redis下载编译

选的版本是5.0.2

先升级下依赖环境

    sudo yum install -y vim ruby rubygems gcc gcc-c++ wget

<!--more-->

下载redis 
    
    wget https://github.com/antirez/redis/archive/5.0.2.tar.gz

解压
    
    tar -zvxf 5.0.2.tar.gz

编译
    
    make


## 配置运行

配置文件

    cd redis-5.0.2/
    vim redis.conf
    把bind从127.0.0.1改为0.0.0.0//允许所有地址连接
    把cluster-enabled从no改为yes//允许集群
    把daemonize从no改为yes//后台运行

运行

    cd src/
    ./redis-server ../redis.conf

可以看到如下说明运行成功

    19070:C 22 Jan 2019 10:18:37.299 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
    19070:C 22 Jan 2019 10:18:37.299 # Redis version=5.0.2, bits=64, commit=00000000, modified=0, pid=19070, just started
    19070:C 22 Jan 2019 10:18:37.299 # Configuration loaded

测试连接

    ./redis-cli -h 127.0.0.1 -p 6379

如果需要退出

    ./redis-cli -p 6379 shutdown
    
## 集群

到src目录运行,这边网上大部分教程用的是redis-trib.rb这个ruby脚本,但是我使用的时候redis要求我用redis-cli

    ./redis-cli --cluster create 192.168.1.150:6379 192.168.1.151:6379 192.168.1.152:6379 192.168.1.153:6379 192.168.1.154:6379 192.168.1.155:6379 --cluster-replicas 1

查看集群信息

    ./redis-cli -h 127.0.0.1 -p 6379
    cluster info

使用 [Redis Desktop Manager ](http://docs.redisdesktop.com/en/latest/install/#build-from-source)测试

可以看到插入一个键后其他节点也同步了

    ![redis集群测试](/blog/img/redis/redis_cluster.png)