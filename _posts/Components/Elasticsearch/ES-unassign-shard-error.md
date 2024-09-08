---
title: ES_unassign_shard_error
date: 2020-05-26 23:16:20
tags: [Elasticsearch, JVM, Debug]
---

# 简述

**问题描述：** ES  环境上的索引`.security-6`(ES用于存储security相关数据的索引，如role definitions, role mappings的等等) **常常会出现分片无法分配的问题**。由于Replication Shard无法分配，会导致集群Yellow.

**解决方案**：加大内存或调整 参数-XX:CMSInitiatingOccupancyFraction的值,来提早触发Major GC，增加MajorGC频次。

**Root Cause**：ES由于内存空间不足，老年代与年轻代同时GC，触发Concurrent Mode Failure, 收集器降级为Serial-old（单线程，效率低）从而导致ES节点失去响应，因此节点离开集群。

当节点回到集群，由于`.security`索引被锁，无法在最大同步时长5S实现同步，导致Shard Allocation Failure.

<!--more-->

# 详细描述

ES是Java程序，当其内存出现不足时，是使用垃圾收集器回收过期对象。

当老年代垃圾收集器CMS在回收过程时，同时又触发了一次Minor GC(年轻代的GC), 此刻由于老年代的堆内存本身已经不足，没有足够的空间存放由MinorGC晋升上来的对象。由此引起了`Concurrent Mode Failure`.

> 2020-05-23T12:13:55.737-0400: 1898982.315: [GC (Allocation Failure) 2020-05-23T12:13:55.738-0400: 1898982.315: [ParNew (promotion failed): 613028K->613028K(613440K), 1.5854459 secs]2020-05-23T12:13:57.323-0400: 1898983.901: [CMS2020-05-
> 23T12:13:57.586-0400: 1898984.164: [CMS-concurrent-preclean: 6.350/14.628 secs] [Times: user=7.81 sys=3.34, real=14.63 secs]
> (**concurrent mode failure**): 7292273K->1576223K(7707072K), 127.3055051 secs] 7860257K->1576223K(8320512K), [Metaspace: 116541K->116541K(1161216K)], 128.9056865 secs] [Times: user=4.62 sys=2.66, real=128.89 secs]
> 2020-05-23T12:16:04.644-0400: 1899111.221: **Total time for which application threads were stopped: 128.9195097 seconds,** Stopping threads took: 0.0115353 seconds

此时JVM将使用`Serial Old`垃圾收集器来临时替换CMS。Serial Old是一款单线程的垃圾收集器，效率低，且工作时会长时间停止用户线程工作。因此节点失去响应时间，超过了节点与集群心跳通信的最大延迟33秒，节点因此暂时被标记为离开集群。

| Setting         | Description                                                  |
| --------------- | ------------------------------------------------------------ |
| `ping_interval` | How often a node gets pinged. Defaults to `1s`.              |
| `ping_timeout`  | How long to wait for a ping response, defaults to `30s`.     |
| `ping_retries`  | How many ping failures / timeouts cause a node to be considered failed. Defaults to `3`. |

当GC完成后，节点再次加入集群，Master节点会同步节点上的Shard信息，进行Shard Rebalance或Shard信息同步等，此刻由于`.security`索引正由于某种原因被锁定，从而导致超过5S的分片最大重试时间，分片分配失败。

> failed to create shard, failure IOException[**failed to obtain in-memory shard lock**]; nested: ShardLockObtainFailedException[[.security-6][0]: obtaining shard lock timed out after 5000ms]; ], allocation_status[no_attempt]]]"

# Reference

[Back up a cluster’s security configuration](https://www.elastic.co/guide/en/elasticsearch/reference/7.x//security-backup.html)

[Zen Discovery](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/modules-discovery-zen.html)