---
title: DB_Raft分布式一致性
date: 2018-07-23 17:47:01
tags: Raft
---

> Raft是一种便于理解和实现的一致性算法，它在容错方面的表现与Paxos相当，不同之处在于它被分解为相对独立的子问题，并且它干净的解决了实际系统中所需的主要部分。

与Paxos相比，Raft更为简洁易懂，只需要读懂几篇论文，就可以相对的实现一个分布式一致性的系统，这也是Raft在工程中被广泛应用的原因。本文就将介绍Raft是如何实现分布式一致性的。

<!-- more -->

------

# Mark

Raft的基本东西已经看完了 就是一下几篇文章  由于工作任务繁重 暂时搁置等待下次来填坑



***

#### 相关资料

[GO_Raft实现](https://github.com/goraft/raft)

https://github.com/coreos/etcd

http://thesecretlivesofdata.com/raft/

https://zhuanlan.zhihu.com/p/27207160

http://www.ywnds.com/?p=8040

https://www.cnblogs.com/mindwind/p/5231986.html

raft paper