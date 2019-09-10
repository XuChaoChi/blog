---
title: Kafka基本概念和操作
tags: [kafka]
date: 2019-08-07 14:28:46
category: kafka
keywords: kafka
---

# Kafka基本概念和操作

### broker(kafka进程)

一个独立的kafka进程被称为broker

- 通过zookeeper查看集群中broker命令:

        $ ./zkCli.sh ls /brokers/ids
        > [1003, 1001, 1002]
        
<!--more-->

- broker配置(server.properties)

|配置|说明|
|-|-|
|broker.id|集群中的唯一id|
|port|监听端口|
|zookeeper.connect|hostname:port/path 后面的path是chroot路径，可以用来备份|
|zookeeper.connect.timeout.ms|连接超时|
|log.dirs|数据存放目录，可以用逗号分割来指定多个，同一分区数据在同一路径,新分区将存到分区最少的路径|
|num.recovery.threads.per.data.dir|恢复数据时每个目录的线程数|
|num.partitions|新主题的分区数|
|log.retention.ms|数据保留时间|
|log.retention.bytes|每个分区最多保留数据大小|
|log.segment.bytes|单个日志关闭大小|
|log.segment.ms|单个日志关闭时间|
|message.max.bytes|单个消失最大关闭时间|
|log.flush.interval.messages|收到多少数据后写入磁盘|
|log.flush.interval.ms|到达多少后写入磁盘|
|offsets.topic.replication.factor|topic的offset的备份数__consumer_offset|
|transaction.state.log.replication.factor|topic的事务状态备份数__transaction_state|
|transaction.state.log.min.isr|暂时不理解|
|num.network.threads|收发消息的线程数|
|socket.send.buffer.bytes|发送缓冲区大小|
|socket.receive.buffer.bytes|接收socket大小防止抛出OOM|

### topic(主题)

生产者往topic里写消息，消费者从读消息

- topic创建命令

        $./kafka-topics.sh --create --topic testname --replication-factor 2 --partitions 2 --zookeeper 127.0.0.1:2181
        >Created topic testname.

- topic自动创建
  - 配置中将auto.create.topics.enable=true,开启后发送消息时如果没有对应的topic则将自动创建

- topic查看

    - 直接使用zk
        
            $./zkCli.sh ls /brokers/topics
            >[testname, test_topic]

    - kafka通过zk
  
            $./kafka-topics.sh --zookeeper 127.0.0.1:2181 --describe --topic testname
            >Topic:testname  PartitionCount:2        ReplicationFactor:2     Configs:
                Topic: testname Partition: 0    Leader: 1001    Replicas: 1001,1002     Isr: 1001,1002
                Topic: testname Partition: 1    Leader: 1002    Replicas: 1002,1003     Isr: 1002,1003

- topic删除
 
     设置 delete.topic.enable=true

        ./kafka-topics.sh --delete --topic testname --zookeeper 127.0.0.1:2181

### partitions(分区)

用来对topic的横向拓展，每个partition是一个有序的队列，将一个topic的数据保存到不同的分区

- 查看分区

    - kafka通过zk
 
            ./zkCli.sh ls /brokers/topics/test_topic/partitions

    - 同样可以使用查看topic的命令查看
  
             $./kafka-topics.sh --zookeeper 127.0.0.1:2181 --describe --topic testname

### Replicas(副本)

一个分区可有一个或多个副本，Leaeder副本用于接收读写请求，Follower副本用于做备份

### 生产者

向kafka写入数据

    kafka-console-producer.sh --broker-list localhost:9092 --topic testname

### 消费者

向leader读取数据，新版本方法

    kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test_topic  --from-beginning