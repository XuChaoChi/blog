---
title: Docker部署kafka
tags: [linux,docker,kafka]
date: 2019-05-22 00:16:46
category: docker,kafka
keywords: linux,docker,kafka
---

# Docker部署kafka

## Dockerfile下载

    git clone https://github.com/wurstmeister/kafka-docker.git

## Compose安装

    sudo curl -L "https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

    sudo chmod +x /usr/local/bin/docker-compose

## 修改配置

进入到kafka-docker目录

    vim docker-compose.yml

将KAFKA_ADVERTISED_HOST_NAME修改成本机地址

<!--more-->

## 使用

初始化
    
    docker-compose up -d

使用docker ps查看的结果

![1](/blog/img/docker/docker_kfk01.png)  

添加broker，__注意scale后一定要用下面的stop和start一下，不然只能运行一个，搞了好久才发现！！__

    docker-compose scale kafka=3

关闭集群

    docker-compose stop

开启集群

    docker-compose start

使用docker ps查看的结果

![1](/blog/img/docker/docker_kfk02.png)  