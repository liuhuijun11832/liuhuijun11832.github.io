---
title: MySql-join实战
categories: 编程技术
date: 2019-06-06 10:57:43
tags: MySql
keywords: [MySql]
description: mysql join语句在开发中的使用。
---
# 简述
Join语句在Mysql多表联查的使用中非常广泛，通常来说，开发人员的意识中都会觉得Join的查询效率比子查询要高，但是如果滥用时，Join的效率并不如人意。


# Index Nested-Loop Join
首先看一个sql语句：
```sql
select * from t1 straight_join t2 on t1.id = t2.id
```
这里直接使用straight_join指定t1作为驱动表，t2作为被驱动表。

执行流程：
1. 从t1中读入一行数据R；
2. 从R中取出a字段去t2表中查找；
3. 取出t2中满足条件的行，跟R组成一行，作为结果集的一部分；
4. 重复执行1~3，直到表t1的末尾循环结束。

在这个流程里：
1. 扫描t1时候时全表扫描n；
2. a字段去t2表中搜索走的树搜索m，总的扫描行数时n+m；

如果将join改成单表查询：
```sql
select * from t1;
select * from t2 where a=$R.a
```
1. 查出t1所有数据n；
2. 循环遍历n行数据；
3. 从每一行取出a，到t2中去查；
4. 把返回的结果和R构成结果集的一行。

扫描行数也是m+n，但是客户端要多发100条语句，同时自己拼接语句和结果。

可以看到在INL中，驱动表都是全表扫描n；被驱动表需要先扫描索引a（假如用到了索引），再搜索主键索引，每一个索引树所花费的时间都是log₂M


