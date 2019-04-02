---
title: Mysql-数据可靠性保证
categories: 学习笔记
date: 2019-03-26 10:26:41
tags: MySql
keywords: [Mysql,binlog,undo log]
description: msyql数据可靠性保证的原理
---

# 说明

这篇笔记记载一下mysql中对于数据的保障机制，我们都知道mysql的redo log和binlog的存在保证了数据的完整性和恢复机制。而undo log没有日志文件，只是一个逻辑日志，本质上undo log也是要借助redo log实现持久化保护。

<!--more-->

# binglog写入机制

binlog的写入机制容易理解，主要有这几步骤：

1. 每一个mysql线程都有自己的binlog cache，这个阶段就是将binlog cache write到磁盘页缓存page cache，但是共用同一份binlog文件；

2. 将page cache中的数据fsync到物理磁盘文件中，这个时候才是真正占磁盘IOPS。

主要是通过`sync_binlog`这个参数来控制：

* `sync_binlog=0`：每次提交事务都只write，而不fsync；

* `sync_binlog=1`：每次提交事务都会执行fsync；

* `sync_binlog>N`：每次wirite，但是累积到N个事务才会fsync。

# redo log写入机制

redo log的写入机制听起来也和binglog差不多，即：

1. redo log存在于进程的物理内存中，名为redo log buffer，首先会将redo log buffer write到磁盘页缓存page cache；

2. fsync到磁盘文件中。

cache和buffer看起来很类似，但是实际上作用是不一样的，cache一般用于读缓存，buffer一般用于写加速。

redo log的写入策略可以通过`innodb_flush_log_at_trx_commit`来控制：

它有三种可能的取值：

* 0：每次事务提交时都只把redo log留在buffer中；

* 1：每次事务提交都将redo log进行持久化，并且在prepare阶段就会进行一次持久化；

* 2：每次事务提交都只是将redo log写到page cache。

>  innodb有一个后台线程，每隔1秒都会执行上面两个步骤（定时轮询）。没有提交的事务的redo log也有可能随着其他正在提交事务的redo log写到磁盘（并行提交）。redo log buffer占用空间到达innodb_log_buffer_size一半的时候也会触发write。

# 整体时序

线上一般会采取“双1”配置，即`sync_binlog`和`innodb_flush_log_at_trx_commit`都是1，此时，binglog和redo log的整体执行时序是这样的：

redo log prepare:write - binlog cache:write - redo log prepare:fsync - binlog cache fsync - redo log commit: write

也就是说在redo prepare阶段的redo log就已经进行了一次持久化操作。前面曾经说过redo log文件是一个环形的，每个write point都会有一个日志逻辑序列号（LSN），它是单调递增的，每次写入redo log都会在LSN上加redo log长度，提交第一个事务的时候，它会将小于LSN（因为提交第一个事务的时候可能其它事务的redo log会继续增长LSN）的事务一起进行fsync。这种组提交的方式可以尽可能节约磁盘的IOPS，也就有了上面的时序，放在binlog cache的write后面来拖时间。同样binlog也能够实现组提交，但是由于redo log prepare:fsync的时间一般比较快，所以binlog的组提交不是非常明显，可以通过`binlog_group_commit_sync_delay`和`binlog_group_commit_sync_no_delay_count`来提升组提交效果，前一个参数表示延迟多少微秒执行fsync，后一个参数表示累积多少次以后执行fsync，两个条件互为或的关系。

# 总结

MySql如果出现IO瓶颈，可以考虑以下三种方法：

1. `binlog_group_commit_sync_delay`和`binlog_group_commit_sync_no_delay_count`减少binlog写盘次数，但是会增加语句响应时间，没有丢失数据的风险；

2. 将`sync_binlog`设置为大于1的值，这样有可能导致丢失部分binlog；

3. 将`innodb_flush_log_at_trx_commit`设置为2，这样有可能会丢失数据。
