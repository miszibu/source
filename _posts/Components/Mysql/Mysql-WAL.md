---
title: Mysql-WAL
date: 2020-08-09 22:30:20
tags: [Mysql]
---

"In computer science, write-ahead logging (WAL) is a family of techniques for providing atomicity and durability (two of the ACID properties) in database systems."——维基百科

<!--more-->

# Introduction

在计算机领域，**WAL（Write-ahead logging，预写式日志）**是数据库系统提供**原子性**和**持久化**的一系列技术。

* **binlog**: 记录数据库执行的写入操作，用于主从同步和数据恢复。

* **redo log:** 实现**持久性**，在执行实际操作前，先将操作写入 redo log。即便系统意外宕机，也能从 redo log 中恢复未执行完操作。
* **undo log**：实现**原子性**，可以通过 undo log 撤销执行完的部分操作

# binlog vs redolog

|          | redo log             | binlog                      |
| -------- | -------------------- | --------------------------- |
| 文件组织 | 循环写，文件大小固定 | 追加写，文件大小可以设置    |
| 实现方式 | innodb 引擎实现      | server 层实现，所有引擎都有 |
| 使用场景 | 崩溃恢复 crash-safe  | 主从复制和数据恢复          |



# binlog
binlog用于记录数据库执行的写入性操作, 是一种逻辑日志。
* **逻辑日志**：可以简单理解为记录的SQL语句
* **物理日志**：因为 mysql 数据最终保存在磁盘页上，物理日志记录的是数据页的变更。

binlog 是追加写入文件，可以设置单个 binlog 文件的大小，当达到最大时，生成新的文件来保存日志。

## binlog使用场景
1. **主从复制**：在Master端开启binlog，然后将binlog发送到slave端，Slave端重放 binlog 从而实现主从一致。
2. **数据恢复**：通过使用mysqlbinlog工具来恢复数据。

## binlog刷盘时间
对于InnoDB存储引擎而言，只有在事务提交时才会记录binlog，此时记录还在内存中。
Mysql通过sync_binlog参数控制binlog的刷盘时机:

* **0:** 由系统自行判断何时写入磁盘
* **1:** 每次commit时将binlog写入磁盘
* **N:** 每N个事务，才会将binlog 写入磁盘

**总结：**sync_binlog 参数为1时最安全，这也是 MySQL5.7.7之后版本的默认值。实际中可以适当调整，以降低fsync频率，牺牲部分一致性来换取更好的性能。


## binlog日志格式
> 在 MySQL 5.7.7之前，默认的格式是STATEMENT，MySQL 5.7.7之后，默认值是ROW。日志格式通过binlog-format指定。

* **STATMENT:**基于SQL语句的复制(statement-based replication, SBR)，每一条会修改数据的sql语句会记录到binlog中。
优点：不需要记录每一行的变化，减少了binlog日志量，节约了IO, 从而提高了性能；
缺点：在某些情况下会导致主从数据不一致，比如执行sysdate()、sleep()等。
* **ROW:** 基于行的复制(row-based replication, RBR)，不记录每条sql语句的上下文信息，仅需记录哪条数据被修改了。
优点：不会出现某些特定情况下的存储过程、或function、或trigger的调用和触发无法被正确复制的问题；
缺点：会产生大量的日志，尤其是alter table的时候会让日志暴涨
* **MIXED**: 基于STATMENT和ROW两种模式的混合复制(mixed-based replication, MBR)，一般的复制使用STATEMENT模式保存binlog，对于STATEMENT模式无法复制的操作使用ROW模式保存binlog



# redo log

数据库需要实现**持久性**: 只要事务成功提交，那么对数据库的修改就会永久性的保存下来。

因此为了实现持久性，最直接的办法就是每次提交成功，就直接将数据页的变动更新到磁盘上去，但这样频繁的磁盘操作严重影响性能。因此提出了 **redo log**， **用于记录事务对数据页做了哪些修改。**

redo log 包括了

* 内存中的 redo log buffer
* 磁盘中的 redo log file

每当 Mysql 执行一条 DML(数据库操作语言)语句，就将计入写入 redo log buffer，后续批量将 buffer 通过 fsync()刷入磁盘中的 redo log file。

在计算机操作系统中，用户空间(`user space`)下的缓冲区数据一般情况下是无法直接写入磁盘的，中间必须经过操作系统内核空间(`kernel space`)缓冲区(`OS Buffer`)。因此，`redo log buffer`写入`redo log file`实际上是先写入`OS Buffer`，然后再通过系统调用`fsync()`将其刷到`redo log file`中，过程如下：

![](./Mysql-WAL/redolog.image)

## redo log 刷盘时机

| 参数值              | 含义                                                         |
| ------------------- | ------------------------------------------------------------ |
| 0（延迟写）         | 事务提交时不会将`redo log buffer`中日志写入到`os buffer`，而是每秒写入`os buffer`并调用`fsync()`写入到`redo log file`中。也就是说设置为0时是(大约)每秒刷新写入到磁盘中的，当系统崩溃，会丢失1秒钟的数据。 |
| 1（实时写，实时刷） | 事务每次提交都会将`redo log buffer`中的日志写入`os buffer`并调用`fsync()`刷到`redo log file`中。这种方式即使系统崩溃也不会丢失任何数据，但是因为每次提交都写入磁盘，IO的性能较差。 |
| 2（实时写，延迟刷） | 每次提交都仅写入到`os buffer`，然后是每秒调用`fsync()`将`os buffer`中的日志写入到`redo log file`。 |

## redo log 记录形式

redo log 采用了循环写的机制，不同 binlog 的AOF机制。当写到结尾时，会回到开头循环写日志。

* **write pos**: redo log 当前写入位置
* **check point**: 已经被持久化的 redo log的位置

顺时针来看，check point-> write pos 之间的 redo log 即为尚未被执行的 redo log。

write pos-> check point 之间的则是已经被执行过，数据变动已经落盘的 redo log，可以理解为空的。

当 write pos追上 check point 时，会先推动 check point 向前移动，释放出位置用于记录新的日志。

![](./Mysql-WAL/redolog_writefile.image)

## redo log 与宕机恢复

当启动 innodb 时，总会进行恢复操作，从 redo log file 文件中恢复出尚未落盘的数据变动。



# Undolog 

[Mysql事务](https://miszibu.github.io/2020/08/13/Components/Mysql/Mysql%E4%BA%8B%E5%8A%A1/)

## Reference

[必须了解的mysql三大日志-binlog、redo log和undo log](https://juejin.cn/post/6860252224930070536#heading-0)

