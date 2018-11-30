---
title: Java_代码规范
date: 2018-06-27 17:38:03
tags: [Java,Mark]

---



在平时的Coding中养成良好的代码规范，会极大的提高个人的代码可读性和可维护性。

本文记录一些平时所遇到的一些规范相关的东西也不仅仅局限于JAVA，主要还是来自于《阿里巴巴JAVA开发手册》里的内容总结。IDEA 已有**阿里代码规范检查插件**，相当不错、

> 规范化的代码是一种美，优秀的程序员不需要靠代码的风格来标新立异。——zibu

<!--more-->

### 编程规范

- 首行四个空格
- if for 单行也需要大括号
- 左起大括号不换行

### 异常日志###

### 单元测试###

### 安全规约###

### MySQL数据库###

### 工程结构###

### 设计规范###

###DO、DTO、BO、AO、VO、POJO定义###

分层领域模型规约：

- DO（ Data Object）：与数据库表结构一一对应，通过DAO层向上传输数据源对象，DAO层是Mapper层的SQL封装。
- DTO（ Data Transfer Object）：数据传输对象，Service或Manager向外传输的对象。
- BO（ Business Object）：业务对象。 由Service层输出的封装业务逻辑的对象。
- AO（ Application Object）：应用对象。 在Web层与Service层之间抽象的复用对象模型，极为贴近展示层，复用度不高。
- VO（ View Object）：显示层对象，通常是Web向模板渲染引擎层传输的对象。
- POJO（ Plain Ordinary Java Object）：在本手册中， POJO专指只有setter/getter/toString的简单类，包括DO/DTO/BO/VO等。
- Query：数据查询对象，各层接收上层的查询请求。 注意超过2个参数的查询封装，禁止使用Map类来传输。

领域模型命名规约：

- 数据对象：xxxDO，xxx即为数据表名。
- 数据传输对象：xxxDTO，xxx为业务领域相关的名称。
- 展示对象：xxxVO，xxx一般为网页名称。
- POJO是DO/DTO/BO/VO的统称，禁止命名成xxxPOJO。

* 
