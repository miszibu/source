---
title: Kafka_QucikStart
date: 2019-03-14 00:15:13
tags: [Kafka]
---

> Kafka 常用命令记录贴

<!--more-->

## Start Server

```shell
# 开启系统集成的Zookeeper
# 简便做法，不推荐
bin/zookeeper-server-start.sh config/zookeeper.properties

# 开启Kafka Broker，使用指定properties
bin/kafka-server-start.sh config/server.properties
```



------



## Topic

```shell
# 创建Topic 并设置命令行设置相关参数
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic ${topicName}

# List Topics
bin/kafka-topics.sh --list --zookeeper localhost:2181

# Get Topic Detail
bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic ${TopicName}
```

------



## Producer||Consumer Command

```shell
# 命令行建立一个Producer
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
# 命令行建立一个Consumer
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic ${topicName} --from-beginning
```

------


