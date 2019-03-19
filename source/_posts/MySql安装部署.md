---
title: MySql安装部署
categories: 编程技术
date: 2019-03-04 15:05:53
tags:
- MySql
keywords: [MySql]
description: MySql在各个环境下的安装部署
---
# 描述
随手记一下mysql在linux和在mac os下的一些安装以及踩过的坑。大版本号是5.7，小版本应该没有太大影响。
<!--more-->
# 安装
## Windows
mysql在windows平台上的安装比较简单，但是最好不要在中文目录下安装比较重要的开发软件，不然出现一些问题真的没地方哭。
 
## MAC OS X
默认安装位置在/usr/local/下，新版数据库没有用户自定义配置文件，可以用vim新建一个my.cnf文件，将/usr/local/mysql/suppoty-files/目录下的my-default.cnf里的内容复制到新建的my.cnf里，假如无法复制，可以在控制台按command+s打印控制台内容，复制到my.cnf里以后，将[mysqld]下添加

	datadir=/usr/local/mysql/data
	basedir=/usr/local/mysql/
	character-set-server=utf8(该段代码可选，设置数据库服务端编码格式)

对于有存储微信表情的需要来说，字符编码建议设置为utf8mb4，该字符集是对utf8的的一种补充。

同时在[mysqld]上添加[client]，在[client]下添加：

	default-character-set=utf8(设置客户端创建数据库默认的编码格式)

修改密码：`set password for root@localhost=password('root')`
假如/etc/my.cnf文件[mysqld] 下加入了`skip-grant-tables`上面这个修改密码的语句就不能使用了，
此时应该使用以下语句：
`update user set password=password('root') where user='root';
`字符集查询结果

```sql
mysql> show variables like '%char%';
+--------------------------+--------------------------------------------------------+
| Variable_name            | Value                                                  |
+--------------------------+--------------------------------------------------------+
| character_set_client     | utf8                                                   |
| character_set_connection | utf8                                                   |
| character_set_database   | utf8                                                   |
| character_set_filesystem | binary                                                 |
| character_set_results    | utf8                                                   |
| character_set_server     | utf8                                                   |
| character_set_system     | utf8                                                   |
| character_sets_dir       | /usr/local/mysql-5.5.23-osx10.6-x86_64/share/charsets/ |
+—————————————+--------------------------------------------------------+
```

## CentOS
使用的是RPM包的方式，下载完mysql的tar包，然后解压开能看到很多mysql-community-开头的.rpm文件，使用如下命令就可以安装`sudo yum install mysql-community-{server,client,common,libs}-* --exclude='*minimal*'` 当然要排除最小安装，然后需要找到临时密码才能进入数据库改密码：`sudo grep 'temporary password' /var/log/mysqld.log`找到密码就可以使用`mysql -uroot -p` 登陆，然后修改密码:
`mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';`

如果需要远程互联网访问的话，需要使用如下语句开启权限：

`mysql> grant all on *.* to 'root'@'%' identified by 'root'；`

**注：除修改密码和查询结果的操作外，其他操作必须在停止数据库的情况下进行。
假如忘记了数据库密码或者不知道数据库密码，可以进入/etc/my.cnf文件[mysqld] 下加入了skip-grant-tables，输入密码的时候就可以不输入直接进入。修改密码以后再取消那段语句。也可以直接使用参数启动mysql的安全模式`/usr/local/mysql/bin/mysqld_safe --skip-grant-tables`，执行`update mysql.user set authentication_string=password('newpassword') where user='root' and Host = 'localhost';`和`flush privileges;`以后再以正常模式启动mysql**

# 使用原则
摘录自互联网《mysql 36军规》：

1. 不在数据库做运算，CPU计算必须移至业务层；
1. 控制单表数量在1000万以内，业内公认mysql单表1000万数据时树的高度在3～5，性能较好；
1. 控制列数量：控制字段数量在20个以内；
1. 平衡范式和冗余：为提高效率牺牲范式设计，冗余数据；
1. 拒绝Big sql，Big transaction，Big batch；
1. 用好数值类型：tinyint(1Byte)，smallint(2Byte)，mediumint(3Byte)，int(4Byte)，bigint(8Byte)，所以如int(1)是不合理的；
1. 字符转化为数字，例如，使用int而不是char(15)存储IP；
1. 优先使用enum或set；
1. 避免使用NULL字段，因为NULL字段很难查询，并且NULL字段的索引需要额外的空间，NULL字段的复合索引无效，所以int类型的字段可以默认为0，varchar类型的字段可以默认为空字符串；
1. 少用text/blog：varchar性能会比text和blog高很多，如果避免不了可以拆表；
1. 不在数据库里存图片；
1. 合理使用索引；
1. 字符字段必须建前缀索引；
1. 不在索引做列运算；
1. innodb主键推荐使用自增列，字符串不应该做主键，如果不指定主键，innodb会使用唯一且非空值索引代替；
1. 不用外键，由程序保证约束；
1. sql语句尽可能简单，一条sql只能在一个cpu运算；
1. 事务尽可能简单；
1. 避免使用函数和触发器；
1. 不使用select *；
1. or改写为in，or的效率是n级别，in的效率是log(n)级别，in的个数控制在200以内；
1. or改写为UNION，mysql的索引合并比较弱智：select id from t where phone='159' or name = 'admin' => select id from t where phone = '159' union select id from t where phone = 'admin';
1. 避免负向%
1. 慎用count(*)（个人不认可）
1. limit分页数据一旦很大，尽量使用索引覆盖；
1. 使用union all 代替union；
1. 少用链接join；
1. 使用group by分组，自动排序；
1. 使用同类型比较；
1. 使用load data 倒数比insert快约20倍；
1. 打散批量更新；
1. 性能分析：explain，show profile，mysqlsla，mysqldumpslow；show slow log；show processlist，show query\_response\_time(percona)

# 常用姿势
## 查询处理
```sql
select distinct * from t left join t1 on t1.t_id = t.id where t.city='beijing' group by 'name' having name='liuhuijun' order by birthday limit 10;
```
如上，那么查询顺序是：
from t,t1  ->  on t1.t_id = t.id ->  where t.city='beijing'  -> group by 'name'  ->  having name = 'liuhuiun'   -> select ->  distinct  *-> order by birthday  ->  limit 10

* from:对from子句种的左表和右表执行笛卡尔积（Cartesian product），产生虚拟表VT1；
* on:对虚拟表VT1进行ON筛选，只有符合t1.t_id = t.id的行才被插入虚拟表VT2；
* join:如果是out join类型，那么保留表中未匹配的行作为外部行添加到虚拟表VT2中，作为VT3，如果有多个连接表，那么再一次执行上面的三步操作；
* where:对虚拟表进行where筛选，只有符合city='beijing'的城市才加入到VT4中；
* group by:根据group by对VT4记录进行分组操作，产生VT5；
* having:对虚拟表VT5进行having筛选，只有符合 name='liuhuijun'的才加入到VT6中；
* select:选择指定的列，生成到虚拟表VT7中；
* distinct:去除重复数据，产生虚拟表VT8；
* order by:将VT8只能够的数据进行排序，产生VT9；
* limit:取出对应条数数据生成VT10，并返回。

可以在建表时使用查询语句：

```sql
<!-- 复制表结构和数据到新表 -->
CREATE TABLE new_table SELECT * FROM old_table
只复制表结构
CREATE TABLE new_table SELECT * FROM old_table where 1 = 2
CREATE TABLE new_table LIKE old_table
 
<!-- 将旧表数据插入新表 -->
insert into new_table select * from old_table
insert into new_table(column1，column2...)  select (dolumn1,column2...) from old_table﻿​
```

## 数据库备份
```sql
<!--备份某数据库到文件-->
mysqldump -uroot -p DB_name > dump_file
<!--备份某数据库到文件，与上面的区别在于这条语句会包含drop table if exist table_name,insert之前会有一个锁表语句lock tables talbe_name write，insert之后会有unlocl tables-->
mysqldump -opt -uroot -p > dump_file
<! --从远程主机的源库传输数据到目标库，前提目标库已经存在-->
mysqldump --host=192.168.1.1 --opt sourceDB | mysql --host=localhost -C targetDB
<!--只备份结构不备份数据-->
mysqldump --no-data --database DB1 DB2 DB3 > dump_file
<!--备份所有表-->
mysqldump --all-databases > dump_file
<!--从dump文件恢复-->
mysql -uroot -p DB_name < dump_file﻿​
```
