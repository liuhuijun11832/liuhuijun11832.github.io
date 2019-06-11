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

可以看到在INL中，驱动表都是全表扫描n；被驱动表需要先扫描索引a（假如用到了索引），再搜索主键索引，每一个索引树所花费的时间都是log₂M，所以总的是`log₂M*2*N+N`，可以看出N对于时间复杂度的影响要更大。

# Simple Nested-Loop Join
```sql
select * from t1 straight_join t2 on (t1.a=t2.b)
```
由于t2的字段b上没有索引，所以不会再走树搜索，而是会全表扫描并且每一都需要做判断，次数为`m*n`，这样效率就会很低，也就是SNLJ算法。

# Block Nested-Loop Join
如果被驱动表t2的条件字段没有索引，那么流程就成了：
1. t1的数据读入线程内存join_buffer中，由于这里是select * ，因此整个t1放入了**内存**；
2. 扫描t2，把表t2中每一行取出来，跟join_buffer中数据做对比，满足join条件的，作为结果集的一部分返回。
由于会对t1和t2都做一次全表扫描，因此总的扫描行数是m+n，对于表t2中的每一行，都需要与t1中的每一行做一次判断，因此总的是`m*n`次判断，但是由于是内存的join_buffer里的判断，所以速度很快，性能很好。

join_buffer的大小是由参数`join_buffer_size`设定的，默认是256k，如果放不下表t1的所有数据就会采取分段放的策略，这样执行过程就变成了：
1. 扫描表t1，顺序读取数据行仿佛join_buffer中，放完某一行join_buffer满了，执行第2步；
2. 扫描表t2，把t2中的每一行取出来，跟join_buffer中数据对比，满足join条件的，作为结果集的一部分返回；
3. 清空join_buffer；
4. 继续扫描表t1，读取第一步该行后面的行放入到join_buffer中，继续执行第2步。

假设驱动表的数据行数是N，需要分K段才能完成算法流程，被驱动表的数据行数是M；N越大，需要分的段数K就越大，所以这里把K表示为`u*N`,那么在这个过程中：
1. 扫描行数是`N+u*N*M`；
2. 内存判断`N*M`次。

结论是：N对时间的影响更大，所以N小一些更好，即驱动表小一些更好。


对于小表的定义：
```sql
select * from t1 straight_join t2 on (t1.b=t2.b) where t2.id<=50;
select * from t2 straight_join t1 on (t1.b=t2.b) where t2.id<=50;
```
第一个语句是t1作为驱动表，但是条件里是t2.id，所以第一个语句t2是小表；
```sql
select t1.b,t2.* from  t1  straight_join t2 on (t1.b=t2.b) where t2.id<=100;
select t1.b,t2.* from  t2  straight_join t1 on (t1.b=t2.b) where t2.id<=100;
```
t2表数据和t1表数据相差不多的情况下，由于t1只查一个字段，而t2需要查所有字段，所以第一个语句使用t2做驱动表时，需要放入join_buffer_size的字段就越多，那么能够放入的数据就越少，所以t1是小表2。

准确说：**决定驱动表的时候，应该是按照两个表各自的条件过滤，过滤完成以后，计算参与join的各字段的总数据量，数据量小的表，就是“小表”。**


# 总结
能够使用索引时，可以使用join；

使用join尽量用小表做驱动表；

不能使用被驱动表索引的语句尽量不要用。