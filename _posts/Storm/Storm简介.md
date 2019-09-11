---
title: Storm简介
date: 2019-03-10 13:52:54
tags: [Storm]
---

> Storm相比于Hadoop等批处理工具，提供了优秀的流计算能力。而且具备Real-TIme处理数据的能力，实时性强的多。比如说Twitter这样的公司，面对大数据流的情况，热搜的数据需要实时的发现，显示。这个时候Storm能起到很好的作用。
>
> 而对一个长期的数据分析，统计而言，无疑Hadoop是更好的选择，同样是大数据处理工具，但侧重点不同。
>
> Storm往往搭配Kafka（MQ)，之所以要搭配消息队列，因为Storm只负责消费数据，数据来源自各个方面，如数据库的日志，Redis的缓存等，这些数据的生产速度与Storm Spout的消费速度是不匹配的。因此需要MQ来缓冲数据。
>
> 当然，为什么是Kafka呢，我觉得Kafka Topic的partition非常适配与Strom的机制。N个Partition映射N个Spout，提高了并发度，增加了处理速度。

<!--more-->

## 基础概念一览

* **Topology**：实时计算应用逻辑的封装，通过Stream Group相互关联的Spout和Blot组成的拓扑结构。
* **Spout:** 数据来源，通过`nextTuple`方法从外部主动获取数据并以Tuple Streamd的形式发送。
* **Blot:** 消费InputStream，产生OutputStream，是Topology中的业务处理模块。
* **Stream**: 数据流指的是在分布式环境中并行创建、处理的一组元组（tuple）的无界序列。
* **Stream groupings**: 数据流分组定义了在 Bolt 的不同任务（tasks）中划分数据流的方式。
* **Nimbus**：管理保存Topology，统筹规划Supervisor资源，合理分配worker进程。
* **Supervisor**: 下载更新Topology，启动监控Worker进程。
* **Worker**: 实际上的Topology执行状态,一个Topology绑定一个worker进程来执行
* **Executor**: Worker进程，启动的Thread。Executor执行Task，Task是由Spout或Blot组成

------



## 基础概念详解







------



## 相关引用

[Apache Storm官方文档](http://storm.apache.org/releases/1.2.2/index.html)

[Apache Storm官方文档中文版——魏勇](http://ifeve.com/apache-storm/)

[storm为什么总是和消息队列一起用呢？](https://www.zhihu.com/question/27836645/answer/38354425)