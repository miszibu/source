---
title: Mysql-索引
date: 2018-06-14 10:01:55
tags: [Mysql,Mark]

---



Mysql在数据库单表上百万之后，性能查询急剧下降，良好的索引可以极大提高表的查询速度。当然主从复制，读写分离来减少单台服务器的压力。SQL优化，磁盘优化等来提高数据查询效率等等都是行之有效的方法。

本文来自无缺哥的[Mysql索引](https://www.processon.com/view/link/5b18e12ce4b001a14d30d9e0)，在个人博客中备份并且增加一些个人的理解。

<!--more-->

### 索引的概述

**什么是索引**：索引是一种**特殊的文件**(InnoDB数据表上的索引是表空间的一个组成部分)，它们包含着对数据表里所有记录的引用指针。更通俗的说，索引也是一张表，该表保存了主键与索引字段，并指向实体表的记录。 

**索引的优点**：

* 通过创建唯一性索引，可以保证数据库表中每一行数据的唯一性
* 可以大大的**加快数据的检索速度**，主要目的
* 可以加速表和表之间的连接，特别是在实现数据的参考完整性方面特别有意义 
* 在使用分组和排序 子句进行数据检索时，同样可以显著减少查询中分组和排序的时间
* 通过使用索引，可以在查询的过程中，使用优化隐藏器，提高系统的性能 

**索引的缺点**：

* **创建索引和维护索引要耗费时间**，这种时间随着数据量的增加而增加
*  索引需要**占物理空间**，除了数据表占数据空间之外，每一个索引还要占一定的物理空间，如果要建立聚簇索引，那么需要的空间就会更大 
* 当对表中的数据进行增加、删除和修改的时候，**索引也要动态的维护**，这样就降低了数据的维护速度 

### 索引的使用

**创建索引**：索引有单列索引和组合索引（一个索引对应几列）

```mysql
ALTER TABLE table_name ADD INDEX index_name (column_list)：添加普通索引，索引值可出现多次。
ALTER TABLE table_name ADD PRIMARY KEY (column_list): 该语句添加一个主键，这意味着索引值必须是唯一的，且不能为NULL。
ALTER TABLE table_name ADD UNIQUE index_name (column_list): 这条语句创建索引的值必须是唯一的（除了NULL外，NULL可能会出现多次）。
ALTER TABLE table_name ADD FULLTEXT index_name (column_list):该语句指定了索引为 FULLTEXT ，用于全文索引。
#栗子
ALTER TABLE testalter_tbl ADD INDEX (c);
```

**删除索引**

```mysql
DROP INDEX [indexName] ON [table_name];
ALTER TABLE [table_name] DROP INDEX [index_name] ;
ALTER TABLE [table_name] DROP PRIMARY KEY ;
#栗子
drop index classify_index on commodity_list ;
```

**查看索引**

```mysql
SHOW INDEX FROM [table_name];
```

https://cloud.tencent.com/developer/article/1004912

https://www.jianshu.com/p/18ab39d8dd88

https://tech.meituan.com/mysql-index.html

https://www.cnblogs.com/alvin_xp/p/4162249.html

https://www.cnblogs.com/tgycoder/p/5410057.html

https://www.processon.com/view/link/5b18e12ce4b001a14d30d9e0