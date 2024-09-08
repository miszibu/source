---
title: Storm简介
date: 2020-05-20 13:52:54
tags: [Storm]
---

> Storm是一个免费开源的**分布式实时计算系统**。利用 Storm 可以很容易做到可靠地处理无限的数据流。
>
> 数据在发往 Storm之前，一般会将数据直接存放到 MQ 中，如 Kafka。Storm 再从Kafka 中读取数据，因为Storm 只负责消费数据，并不提供数据缓存。
>
> 相比于 Hadoop，Storm 更注重数据的**实时性**，而 Hadoop 则更侧重于批处理，各有优点。

<!--more-->

## 基础概念一览

* **Nimbus**：Master 节点，负责分发 topology 给 supervisor

  * Storm 集群的 Master节点，负责管理分发Topology，并指派给具体的 Supervisor 节点上的 worker 进程，由 Worker来运行 Topology 的Spout 或者 Bolt 的 Task.

* **Supervisor**: Slave 节点，负责管理 Worker 进程的生命周期

* **Worker**: 具体处理组件逻辑的**进程**

  * Worker运行的任务只有两种类型，一种是 Spout，另一种是 Bolt。

* **Task:** 逻辑上的一个概念，worker 上运行的 每一个Spout或Bolt 被称为一个 task。 Task不和线程一一对应，一个 executor 可能会运行不同的 task。

* **Executor**: Worker 进程所启动的**线程**， 用于执行 Task。

* **Topology**：一个真正运行的，通过Stream Group相互关联的Spout和Blot组成的拓扑结构。

  * **Spout:** 数据来源，通过`nextTuple`方法从外部主动获取数据并以Tuple Stream的形式发送。
  * **Bolt:** 消费InputStream，产生OutputStream，是Topology中的业务处理模块。
  * **数据模型**：Spout->Tuple->**Bolts**, 数据流从 spout 开始，以 tuple 的形式发送到 bolt，一个 bolt 可以接入多个 Spout/Blot

* **Stream**: tuple 的集合，流动的 tuple

* **Stream groupings**: 数据流分组，定义了在 Bolt 的不同任务（tasks）中划分数据流的方式。

  

------



## 基础概念详解







------



## 相关引用

[Apache Storm官方文档](http://storm.apache.org/releases/1.2.2/index.html)

[Apache Storm官方文档中文版——魏勇](http://ifeve.com/apache-storm/)

[storm为什么总是和消息队列一起用呢？](https://www.zhihu.com/question/27836645/answer/38354425)