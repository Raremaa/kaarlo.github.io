---
title: "服务PostgreSQL迁移至MySQL全记录"
date: 2021-06-17 09:51:31.636
draft: false
type: "post"
showTableOfContents: true
tags: ["MySQL","PostgreSQL","RDBMS"]
---

# 1. 语法差异

这里总结一下这次迁移过程中遇到的语法差异：

- PG中，`select [column1] || [column2]` 用来连接字符串；MySQL中 || 是逻辑运算符，按照PG语法，需要改成字符串连接函数 `select concat([column1], [column2])`
- PG支持boolean，MySQL没有该类型，需要转换为bit(1)
- PG中的关键字转义用的`""`，需要替换为MySQL支持的转义符（可能关键字也有差异，一些MySQL的非关键字我看之前同事在PG中也进行了转义，这一点没有去查详细差异）
- PG支持window function，Mysql5.7（MySQL 8.0支持）尚未支持。
- 部分PG的语法在MySQL中可能需要嵌套子查询

# 2. 迁移工具选型

基本考虑阿里云的线上工具，这里做一个DTS和DataWorks的比较： 

## 2.1. 阿里云DTS

**DTS可以做到不停机迁移**，**支持全量 + 增量**两种方式。 不停机迁移(不准确)其实是基于全量 + 增量完成，增量迁移不会锁表：

![%E6%9C%8D%E5%8A%A1PostgreSQL%E8%BF%81%E7%A7%BB%E8%87%B3MySQL%E5%85%A8%E8%AE%B0%E5%BD%95%20c1aa24eb091540f4a34e9a51b946a35e/Untitled.png](https://img.masaiqi.com/Untitled.png)

[https://developer.aliyun.com/article/57626](https://developer.aliyun.com/article/57626)

这种迁移方式，迁移流程是这样的：

![%E6%9C%8D%E5%8A%A1PostgreSQL%E8%BF%81%E7%A7%BB%E8%87%B3MySQL%E5%85%A8%E8%AE%B0%E5%BD%95%20c1aa24eb091540f4a34e9a51b946a35e/Untitled%201.png](https://img.masaiqi.com/Untitled1.png)

也就是: 全量数据迁移 -> 增量迁移 -> 业务停服等待增量数据追平 -> 业务开服

这个方案下，**数据迁移是一个“渐进式”的过程，停服时间取决于增量数据的同步时间 + 业务服务的部署时间**。 但是，需要注意，**同数据量下DTS的数据迁移会“慢一点”，**这个是因为原理上它需要对数据库日志进行解析 + 回放的过程： > DTS在全量迁移之前，会先在后台启动一个增量日志拉取及解析程序。这个程序会实时获取源数据库在全量迁移过程中产生的任何增量日志，解析并将其封装为DTS自己的数据格式存储在本地存储系统中。当全量迁移完成后，增量数据回放模块，会去拉取模块中读取存储的增量日志数据，然后通过解析、过滤、封装等步骤，最终拼装成要回放的SQL语句，回放到目标数据库，从而实现源数据库同目标数据库之间的增量数据实时同步。

## 2.2. 阿里云DataWorks

这个其实就是设置两个数据源的表，字段与字段之间设置映射关系，之后进行全量数据迁移。

![%E6%9C%8D%E5%8A%A1PostgreSQL%E8%BF%81%E7%A7%BB%E8%87%B3MySQL%E5%85%A8%E8%AE%B0%E5%BD%95%20c1aa24eb091540f4a34e9a51b946a35e/Untitled%202.png](https://img.masaiqi.com/Untitled2.png)

在这个方案下，流程是这样的：

![%E6%9C%8D%E5%8A%A1PostgreSQL%E8%BF%81%E7%A7%BB%E8%87%B3MySQL%E5%85%A8%E8%AE%B0%E5%BD%95%20c1aa24eb091540f4a34e9a51b946a35e/Untitled%203.png](https://img.masaiqi.com/Untitled3.png)

也就是： 业务停服 -> 全量数据迁移 -> 业务开服

这个方案下，**停服时间是取决于全量数据迁移的时间 + 业务服务的部署时间**  但是，**需要注意，同数据量下DataWorks应该会“快一点”，因为他可以直接读取源数据库数据直接插入目标数据库，不需要做日志的分析处理。** **​**

**在实际上线过程中发现坑点，同步时出现部分数据同步失败，由于一开始我是整个一起执行无法看到日志，建议执行时一个一个表点进去执行，下面会有个控制台输出日志，有错误可以及时定位。建议无论选用哪种方式，都要对数据做一个基本校验（我这里选择数据行数校验，两个表的数据行数一致认为同步成功。）**