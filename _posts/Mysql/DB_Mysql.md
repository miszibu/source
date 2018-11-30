## Not Only Sql数据库

非关系型数据库,这其实是个误称。实际上应该叫做超Sql数据库，NOSQL删除了传统SQL中ACID特性，既**原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）、持久性（Durability）**，从而提高了程序的效率。因此受到了互联网公司的追捧，然后在正规的事务处理中，传统的关系型数据库的事务性仍然是必不可少的。

<!--more-->

## 语法规范

关键词大写，为了在长SQL时保持较好的可阅读性。实际上SQL语句对大小写并不敏感。

## Sql语句

**Database**

```mysql
CREATE DATABASE databaseName;//创建数据库

SHOW DATABASES;//查询数据库列表

USE DBName;//选择数据库

DROP databaseName;//丢掉数据库
```

**Table**

```mysql
CREATE TABLE TBName(name1 type1 NOT NULL PRIMARY KEY AUTO_INCREMENT,...);//创建表

USE TBName；//选择表

SHOW TABLES;//查看所有表

DROP TABLE TBName;//删除数据表

DELETE FROM TBName;//清空表中记录

DESC TBName;
DESCRIBE TBName;
SHOW COLUMNS FROM TBName;//查看User的表结构

ALTER TABLE pet ADD id INT NOT NULL PRIMARY KEY AUTO_INCREMENT FIRST;//增加表的一列 
ALTER TABLE pet DROP COLUMN des;//删除表的一列

ALTER TABLE TBName PRIMARY KEY(colName);//将表的一行设为主键(主键不可重复)
ALTER TABLE TBName DROP PRIMARY KEY(colName);//将表的一行删除主键

RENAME TABLE pet TO animal;//变更表名
```

**索引**

```mysql
CREATE INDEX idxName ON TBName(colName);//为一行设置索引(索引名字UNIQUE)
DROP INDEX idxName ON TBName;//删除索引
注：索引是不可更改的，想更改必须删除重新建。
```

**基本语法**

```mysql
查找：select * from table1 where field1 like ’%value1%’ //使用like来做相似查找
插入：insert into table1(field1,field2) values(value1,value2)
删除：delete from table1 where 范围
更新：update table1 set field1=value1 where 范围
排序：select * from table1 order by field1,field2 [desc]
总数：select count(*) as totalcount from table1
求和：select sum(field1) as sumvalue from table1
平均：select avg(field1) as avgvalue from table1
最大：select max(field1) as maxvalue from table1
最小：select min(field1) as minvalue from table1[separator]
```

**进阶**

~~~Mysql
Mysql 进阶篇 先挖个坑 等工作上有需要了 来填 基础的功能已经够工作使用了
~~~





**视图**

```mysql
Mysql视图是一个虚拟表，其内容由查询定义。同真实的表一样，视图包含一系列带有名称的列和行数据。但是，视图并不在数据库中以存储的数据值集形式存在。行和列数据来自由定义视图的查询所引用的表，并且在引用视图时动态生成。

视图是存储在数据库中的查询的sql 语句，它主要出于两种原因：安全原因，视图可以隐藏一些数据，如：社会保险基金表，可以用视图只显示姓名，地址，而不显示社会保险号和工资数等，另一原因是可使复杂的查询易于理解和使用。

MySQL视图：查看图形或文档的方式。

视图是从一个或多个表或视图中导出的表，其结构和数据是建立在对表的查询基础上的。和表一样，视图也是包括几个被定义的数据列和多个数据行，但就本质而言这些数据列和数据行来源于其所引用的表。
```

## 运维/安全

**备份**：

## 索引

## 优化

**SELECT *降低效率**：

* 查询效率：select * 会在一定程度上增加查询时间。
* 传输效率：从SQL服务器传输到本机上，数据越多会延长时间。

## 相关引用

[MySQL 对于千万级的大表要怎么优化？](https://www.zhihu.com/question/19719997)

[常用SQL命令](https://www.cnblogs.com/0351jiazhuang/p/4530366.html)