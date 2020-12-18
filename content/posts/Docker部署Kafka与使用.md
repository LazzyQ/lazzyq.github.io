---
title: "Docker部署Kafka与使用"
date: 2020-11-28T16:37:06+08:00
publishDate: 2020-11-29T00:41:06+08:00
draft: false
original: true
categories: 
  - Docker
tags: 
  - Docker
  - Kafka
  - 消息队列
---

最近在做日志平台，第一步就是日志收集，使用的filebeat收集日志推送到kafka集群。所以需要搭建Kafka集群，这里记录使用Docker搭建Kafka集群的过程，并对Kafka进行一些常规的操作。

### 搭建ZK集群

之前搭建过ZK集群，这里直接把docker-compose.yml文件贴上

<!--more-->

```yaml
version: "3.8"
services:
  zoo1:
    image: zookeeper:latest
    hostname: zoo1
    container_name: sx_zoo1
    # ports:
    #  - "2181:2181"
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=0.0.0.0:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181
    networks:
      - sx-net
  zoo2:
    image: zookeeper:latest
    hostname: zoo2
    container_name: sx_zoo2
    # ports:
      # - "2182:2181"
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=0.0.0.0:2888:3888;2181 server.3=zoo3:2888:3888;2181
    networks:
      - sx-net
  zoo3:
    image: zookeeper:latest
    hostname: zoo3
    container_name: sx_zoo3
    # ports:
    #   - "2183:2181"
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=0.0.0.0:2888:3888;2181
    networks:
      - sx-net
networks:
  sx-net:
    external: true
```

### 搭建Kafka集群

```yaml
version: '3.8'
services: 
  kafka1: 
    image: wurstmeister/kafka
    container_name: sx-kafka1
    hostname: kafka1
    environment: 
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://:9092 # 与listeners功能相似，用户IaaS环境，ADVERTISED_LISTENERS绑定公网IP，LISTENERS绑定私网IP
      KAFKA_LISTENERS: PLAINTEXT://:9092 # 对外提供服务的入口
      KAFKA_BROKER_ID: 0 # broker的编号，如果集群中有多个broker，则每个broker的编号需要设置不同
      KAFKA_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181 # zk地址
    networks: 
      - sx-net
  kafka2: 
    image: wurstmeister/kafka
    container_name: sx-kafka2
    hostname: kafka2
    environment: 
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://:9092
      KAFKA_LISTENERS: PLAINTEXT://:9092
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181
    networks: 
      - sx-net
  kafka3: 
    image: wurstmeister/kafka
    container_name: sx-kafka3
    hostname: kafka3
    environment: 
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://:9092
      KAFKA_LISTENERS: PLAINTEXT://:9092
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181
    networks: 
      - sx-net
  kafka-manager:
    container_name: sx-kafka-manager
    image: sheepkiller/kafka-manager              ## 镜像：开源的web管理kafka集群的界面
    environment:
        ZK_HOSTS: zoo1:2181,zoo2:2181,zoo3:2181
    ports:
      - "9000:9000"                               
    networks: 
      - sx-net
networks:
  sx-net:
    external: true
```

### 使用Kafka

创建topic

```
kafka-topics.sh --create --zookeeper zoo1:2181,zoo2:2181,zoo3:2181 --replication-factor 2 --partitions 3 --topic beats
```

topic详情
```
kafka-topics.sh --describe --zookeeper zoo1:2181,zoo2:2181,zoo3:2181  --topic beats
```


查看topic列表
```
kafka-topics.sh --list --zookeeper zoo1:2181,zoo2:2181,zoo3:2181 
```

删除topic
```
kafka-topics.sh --delete --zookeeper zoo1:2181,zoo2:2181,zoo3:2181  --topic beats
```

消费消息
```
kafka-console-consumer.sh --bootstrap-server sx-kafka1:9092,sx-kafka2:9092,sx-kafka3:9092 --topic beats
```

生产消息

```
kafka-console-producer.sh --bootstrap-server sx-kafka1:9092,sx-kafka2:9092,sx-kafka3:9092 --topic beats
```

