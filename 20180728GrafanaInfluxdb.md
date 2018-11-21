---
title: 利用InfluxDB和grafana搭建监控服务
tags:
  - InfluxDB
  - grafana
keywords: 'InfluxDB,grafana'
category: 环境搭建
abbrlink: a690ac29
date: 2018-07-28 22:48:33
---

# 利用InfluxDB和grafana搭建监控服务 #

## InfluxDB ##

### InfluxDB介绍 ###

![](https://www.cxc233.com/blog/img/20180728/00.png)

InfluxDB是一个开源的时间序列数据库，没有任何的外部依赖，非常适合用来记录指标，事件和执行分析。有以下特点：
	
- 使用HTTP API 只要发送Http请求就行了，可以不用特地的写一个服务端。
- 可以标记数据，从而实现非常灵活的查询。
- 查询语句像SQL语句，学习成本很低。
- 易于安装和管理，并可快速获取数据。
- 它旨在实时回答查询。这意味着每个数据点在进入时都会被编入索引，并且在返回<100ms的查询中立即可用。

[github地址](https://github.com/influxdata/influxdb)

<!-- more -->

### InfluxDB搭建（centos为例） ###

1. 下载二进制文件
	 
		wget https://dl.influxdata.com/influxdb/releases/influxdb-1.6.0.x86_64.rpm

2. 本地安装
	
		sudo yum localinstall influxdb-1.6.0.x86_64.rpm

3. 开始、重启关、闭服务

		systemctl start influxdb
		systemctl restart influxdb
		systemctl stop influxdb

4. 新建db
		
		curl -XPOST "http://localhost:8086/query" --data-urlencode "q=CREATE DATABASE mydb"

5. 插入数据

		curl -XPOST "http://localhost:8086/write?db=mydb" \
		-d 'cpu,host=server01,region=uswest load=42 1532600667000000000'

		curl -XPOST "http://localhost:8086/write?db=mydb" \
		-d 'cpu,host=server02,region=uswest load=78 1532610667000000000'

		curl -XPOST "http://localhost:8086/write?db=mydb" \
		-d 'cpu,host=server03,region=useast load=15.4 1532630667000000000'
	
	
6. 查询数据

		curl -G "http://localhost:8086/query?pretty=true" --data-urlencode "db=mydb" \
		--data-urlencode "q=SELECT * FROM cpu WHERE host='server01' AND time < now() - 1d"

7. 分析数据

		curl -G "http://localhost:8086/query?pretty=true" --data-urlencode "db=mydb" \
		--data-urlencode "q=SELECT mean(load) FROM cpu WHERE region='uswest'"

----------

## grafana ##

### grafana介绍 ###


Grafana是Graphite，Elasticsearch，OpenTSDB，Prometheus和InfluxDB功能丰富的开源图形分析项目。本次博客就是连接的InfluxDB。

[github地址](https://github.com/grafana/grafana)

### grafana搭建（centos为例） ###

1. 下载二进制文件

		wget https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana-5.2.2-1.x86_64.rpm 

2. 本地安装

		sudo yum localinstall grafana-5.2.2-1.x86_64.rpm

3. 开始、重启关、闭服务

		systemctl start grafana-server
		systemctl restart grafana-server
		systemctl stop grafana-server

4. 本地访问测试（默认账号admin，密码admin，第一次登陆会要求重置密码

		http://127.0.0.1:3000/login

5. 修改密码

	忘记密码的情况：

		grafana-cli admin reset-admin-password 123456

	如果提示Could not find config defaults, make sure homepath command line parameter is set or working directory is homepath测使用：

		grafana-cli admin reset-admin-password --homepath "/usr/share/grafana" 123456

	知道密码的情况：

		curl -X PUT -H "Content-Type: application/json" -d '{
  		"oldPassword": "admin",
  		"newPassword": "newpass",
  		"confirmNew": "newpass"
		}' http://admin:admin@<your_grafana_host>:3000/api/user/password
5. 其他配置可以访问[grafana文档](http://docs.grafana.org/installation/)


## 使用 ##
1. 打开登录网页http://127.0.0.1:3000/login

	![](https://www.cxc233.com/blog/img/20180728/0.jpg)

2. 添加db连接
	![](https://www.cxc233.com/blog/img/20180728/1.jpg)

 	类型选择influxDB，然后填上地址，和账号密码。

	![](https://www.cxc233.com/blog/img/20180728/2.jpg)

	提示Data source is working就可以了

	![](https://www.cxc233.com/blog/img/20180728/3.jpg)

3. 创建图表

	![](https://www.cxc233.com/blog/img/20180728/4.jpg)

4. 配置图表

	![](https://www.cxc233.com/blog/img/20180728/5.jpg)

5. 选择条件

	![](https://www.cxc233.com/blog/img/20180728/6.jpg)

6. 选择时间

	![](https://www.cxc233.com/blog/img/20180728/7.jpg)

7. 结果

	![](https://www.cxc233.com/blog/img/20180728/8.jpg)
