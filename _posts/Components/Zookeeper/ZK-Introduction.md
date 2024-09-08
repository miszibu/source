---
title: ZK-Introduction
date: 2020-08-03 23:34:44
tags: [Zookeeper]
---

Zookeeper基础概念介绍.

ZK中维护了一个分布式的文件系统，每个节点称为Znodes，



---



<!--more-->

## ZK 数据模型

Zk 有一个层级的命名空间，很像一个分布式的文件系统。但是ZK中的每一个Node都可以有自己的属性。可以理解为传统的文件系统中，文件夹也是文件，也可以存放数据。

在创建ZK Node时，有以下限制

* 部分 字符 不可用
* "." 可以使用在节点名称中，但是不能单独使用或多个点一起使用。/a/b.c/d 是允许的，/a/./b 和 /a/../b都是不允许的
* ”zookeeper“已经默认被使用

### ZNodes

每个ZK tree中的节点被称为`ZNode`。 `ZNode`维护了其当前的状态，它包含`Version`，`acl changes`，Timestamp等数据。其中的`version`和Timestamp变量，则提供了ZK验证缓存和协调更新的能力。

每当ZNode数据变动时，就增加数据版本的值。CAS操作会存在ABA问题，带版本的数据则不会存在ABA问题。

当更新操作到达时，会匹配其数据和版本和当前实际的值是否一致，如果不一致，更新操作会失败。

ZNodes是开发者接触的主要实体，它还具备以下的特性。

#### Watchs | 监控节点变动

客户端可以在Znodes上设置`watcher`, ZNode上的数据变动会**触发并清空**`Watcher`，然后给客户端发送提醒。

#### Data Access | 权限控制和数据限制

读操作会读取ZNode上的所有数据，写操作会覆盖ZNode上的所有数据。

每个ZNode都有Access Control List(ACL)来控制用户访问权限。

ZNode不建议存放比较大的数据，客户端和服务端都会检测并**限制数据小于1MB**，但我们存放的数据应该远小于这个数值才好。当我们存放过大的数据时，数据会需要从存储媒介和网络中进行移动，因此操作的延时会增加。如果真的有需要存储一个分布式协调的大数据，可以存储到其他的DB中，在ZK中只存储对应的指针。

#### Ephemeral Nodes | 临时节点

ZK提供了临时节点的概念，这种节点会在会话结束时被移除。临时节点不可拥有子节点，可以使用`getEphemeral()`来获取全部临时节点。

#### Sequence Nodes -- Unique Naming | 顺序Node, 以Unique Number结尾

ZK允许在创建ZNodes时，给予其一个**单调递增的唯一数字ID**作为结尾。这个ID是10位数字，0作为填充，比如xxxxx-0000000001。

> Note: 当前最大的计数器是由其父ZNode保存的，这是一个4Byte长的Int类型，存在OverFlow的可能性。

### Container Nodes (Added in 3.6.0)

当Container Nodes的所有子节点都被删除后，Container Nodes就会在未来的某一时刻被Server删除。

因此当我们在Conainer Nodes中创建节点时，随时有可能遇到`  KeeperException.NoNodeException ` ，当遇到该异常时，需要重新创建Container Node。

>  Container znodes are special purpose znodes useful for recipes such as leader, lock, etc. 

### TTL Nodes (Added in 3.6.0)

TTL Nodes是Container Nodes的基础上又增加了TTL属性的节点。当一个TTL节点没有子节点或者在TTL时间内没有被修改，则节点就会在未来的某一时刻被Server删除。

> TTL Nodes默认关闭，需要修改设置激活，否则会抛出 KeeperException.UnimplementedException. 



### Time In Zookeeper

ZK以多种方式来跟踪时间的变动：

* **Zxid: (ZK Transaction id)** 每一个对于ZK 状态的变更都有一个Zxid。通过比较Zxid的大小就可以获得变更操作的顺序。
* **Version Number**: 节点上的每一个变更都会引起节点Version的变动。
  * version: Znode数据的变更
  * cverion: ZNode子节点的变动
  * aversion: Znode ACL的变动
* **Ticks:** 当使用ZK集群时，ZK Server之间使用Ticks的值作为很多事件的基础单位，比如说会话过期的最小时间为一个Tick, 比如集群状态更新，Peer Server连接超时等等。
* **Real Time**:  ZK基本不使用真实时间，除了在Znode创建和修改时候插入Stat Structure中的timestamp.

### Zookeeper Stat Structure

The Stat structure for each znode in ZooKeeper is made up of the following fields:

- **czxid** The zxid of the change that caused this znode to be created.
- **mzxid** The zxid of the change that last modified this znode.
- **pzxid** The zxid of the change that last modified children of this znode.
- **ctime** The time in milliseconds from epoch when this znode was created.
- **mtime** The time in milliseconds from epoch when this znode was last modified.
- **version** The number of changes to the data of this znode.
- **cversion** The number of changes to the children of this znode.
- **aversion** The number of changes to the ACL of this znode.
- **ephemeralOwner** The session id of the owner of this znode if the znode is an ephemeral node. If it is not an ephemeral node, it will be zero.
- **dataLength** The length of the data field of this znode.
- **numChildren** The number of children of this znode.

## ZK Sessions



## ZK Watches



## 一致性保证

ZK是一个高性能，可伸缩的服务。对于读写操作，反应都很快，但是读操作时快于写操作的。这是因为读操作，ZK可能会返回非最新的数据，这是ZK的一致性保证决定的。

* 顺序一致性：客户端的所有请求都会以它们发送的顺序被执行。
* 原子性
* Single System Image: 在ZK Cluster中，客户端无论连接哪个ZK Server 都可以得到相同的结果。
* 可靠性：一旦更新被应用了，数据会被持久化到下一次覆盖操作之前，这可以引申以下两个推论
  * If a client gets a successful return code, the update will have been applied. On some failures (communication errors, timeouts, etc) the client will not know if the update has applied or not. We take steps to minimize the failures, but the guarantee is only present with successful return codes. (This is called the *monotonicity condition* in Paxos.)
  * Any updates that are seen by the client, through a read request or successful update, will never be rolled back when recovering from server failures
*  *Timeliness* : The clients view of the system is guaranteed to be up-to-date within a certain time bound (on the order of tens of seconds). Either system changes will be seen by a client within this bound, or the client will detect a service outage. 

 Using these consistency guarantees it is easy to build higher level functions such as leader election, barriers, queues, and read/write revocable locks solely at the ZooKeeper client (no additions needed to ZooKeeper). See [Recipes and Solutions](https://zookeeper.apache.org/doc/current/recipes.html) for more details. 

## Reference

[Official ZK Introduction](https://zookeeper.apache.org/doc/current/zookeeperProgrammers.html#_introduction)