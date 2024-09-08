---
title: DDD-Doamin Drive Design
date: 2021-05-26 13:39:04
tags:
---


TODO

<!--more-->

我的理解：传统的开发模式，第一步是设计数据库表结构，Services层处理逻辑，最后通过SQL将数据持久化到数据库，我们所做的一切开发都是以数据库为核心的，Java对象只是数据库某条记录全部或部分的数据。

而 DDD开发模式，则是以Domain Object为核心，Domain Object 内定义自己相关方法(Single Responsibility)，Service 来调用各个 Domain 对象自身的各个方法，并不加上或加上少量逻辑。
