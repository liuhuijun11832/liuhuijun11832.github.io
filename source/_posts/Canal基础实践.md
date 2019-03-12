---
title: Canal基础实践
categories: 编程技术
date: 2019-03-12 14:55:54
tags:
- MySql
keywords: [MySql,Canal]
description: 阿里巴巴MySql增量日志解析组件，提供了增量数据消费等功能
---
# 简述
Canal是一个MySql增量日志解析组件，其原理是将自己伪装成一台MySql Slave机器发送dump请求给Master，然后Master推送binary log给Canal，Canal会解析bin log对象的byte流。Canal提供一系列接口由客户端对增量数据进行自己的业务处理。

<!--more-->
![](Canal基础实践/687474703a2f2f646c2e69746579652e636f6d2f75706c6f61642f6174746163686d656e742f303038302f333130372f63383762363762612d333934632d333038362d393537372d3964623035626530346339352e6a7067.jpeg)

# 数据库配置
## 开启binlog
MySql当前版本为5.7.15，首先需要开启mysql的binlog，因为mysql默认是不开启binlog的。
	
**如何查看MySql是否开启了binlog？**

首先需要`mysql -uroot -p`进入mysql的客户端命令行，然后执行：

```sql
mysql> show variables like 'log_%';
+----------------------------------------+----------------------------------------+
| Variable_name                          | Value                                  |
+----------------------------------------+----------------------------------------+
| log_bin                                | ON                                     |
| log_bin_basename                       | /usr/local/mysql/data/mysql-bin        |
| log_bin_index                          | /usr/local/mysql/data/mysql-bin.index  |
| log_bin_trust_function_creators        | OFF                                    |
| log_bin_use_v1_row_events              | OFF                                    |
| log_builtin_as_identified_by_password  | OFF                                    |
| log_error                              | /usr/local/mysql/data/mysqld.local.err |
| log_error_verbosity                    | 3                                      |
| log_output                             | FILE                                   |
| log_queries_not_using_indexes          | OFF                                    |
| log_slave_updates                      | OFF                                    |
| log_slow_admin_statements              | OFF                                    |
| log_slow_slave_statements              | OFF                                    |
| log_statements_unsafe_for_binlog       | ON                                     |
| log_syslog                             | OFF                                    |
| log_syslog_facility                    | daemon                                 |
| log_syslog_include_pid                 | ON                                     |
| log_syslog_tag                         |                                        |
| log_throttle_queries_not_using_indexes | 0                                      |
| log_timestamps                         | UTC                                    |
| log_warnings                           | 2                                      |
+----------------------------------------+----------------------------------------+
21 rows in set (0.01 sec)
```
log_bin为on即代表开启了binlog，basename代表日志名称和路径。

如果没有开启，需要配置my.cnf文件，Linux/MAC系统下的路径为`/etc/my.cnf`，Windows系统下路径为`C:\ProgramData\MySQL\MySQL Server 5.7\my.ini`，Linux/MAC系统下如果没有这个文件，可以新建一个或者从MySql的安装目录下的support-files里找到名为my-default.cnf文件复制到/etc/下，新增以下配置：


```properties
log-bin=mysql-bin #binlog的名称
binlog-format=ROW #推荐使用二进制记录
expire_logs_days=5 #日志过期时间-5天
server-id=1
binlog-do-db= test #可选，表示记录某个特定库的binlog，如果多个库这个配置要有多个
```
	
## 配置用户
如果想方便省事，可以跳过这步，无需为canal专门配置一个用户。

这里为canal专门新增一个用户，比较安全：

```sql
--创建用户
CREATE USER canal IDENTIFIED BY 'canal';
--赋权--
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
-- GRANT ALL PRIVILEGES ON *.* TO 'canal'@'%' ;
--刷新权限
FLUSH PRIVILEGES;
```
> 扩展：grant命令用法：`grant 权限 on 数据库对象 to 用户`

如果想更改其他用户密码：

```sql
update mysql.user set authentication_string=password('qingyun1') where user='root' and Host = 'localhost';

flush privileges;

--如果mysql提示需要安全模式启动，参考以下--
--首先停止服务
/usr/local/mysql/bin/mysqld stop
--使用安全模式启动数据库
/usr/local/mysql/bin/mysqld_safe --skip-grant-tables
--使用root登陆
/usr/local/mysq/bin/mysql -uroot -p
--修改密码
--正常重启
/usr/local/mysql/bin/msqld restart
```

重启数据库，可以使用上面的命令查看是否开启binlog。

查看当前的binglog文件名称以及作用的数据库：

```sql
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000333 |     1989 | ssm_employee |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```
# Canal部署安装
传送门：[https://github.com/alibaba/canal/](https://github.com/alibaba/canal/)

解压开，然后进入conf目录，一般情况下没有用到集群的时候，canal.propertie可以无需配置，只需配置instance.properties文件即可.


```properties
#################################################
## mysql serverId , v1.0.26+ will autoGen
# canal.instance.mysql.slaveId=0

# enable gtid use true/false
canal.instance.gtidon=false

# position info
# 数据库地址和端口
canal.instance.master.address=192.168.15.107:3306
canal.instance.master.journal.name=
canal.instance.master.position=
canal.instance.master.timestamp=
canal.instance.master.gtid=

# rds oss binlog
canal.instance.rds.accesskey=
canal.instance.rds.secretkey=
canal.instance.rds.instanceId=

# table meta tsdb info
canal.instance.tsdb.enable=true
#canal.instance.tsdb.url=jdbc:mysql://127.0.0.1:3306/canal_tsdb
#canal.instance.tsdb.dbUsername=canal
#canal.instance.tsdb.dbPassword=canal

#canal.instance.standby.address =
#canal.instance.standby.journal.name =
#canal.instance.standby.position =
#canal.instance.standby.timestamp =
#canal.instance.standby.gtid=

# username/password
canal.instance.dbUsername=root
canal.instance.dbPassword=123456
canal.instance.connectionCharset = UTF-8
canal.instance.defaultDatabaseName =
# enable druid Decrypt database password
canal.instance.enableDruid=false
#canal.instance.pwdPublicKey=MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBALK4BUxdDltRRE5/zXpVEVPUgunvscYFtEip3pmLlhrWpacX7y7GCMo2/JM6LeHmiiNdH1FWgGCpUfircSwlWKUCAwEAAQ==

# table regex
canal.instance.filter.regex=
# table black regex
canal.instance.filter.black.regex=

# mq config
canal.mq.topic=example
# dynamic topic route by table regex
#canal.mq.dynamicTopic=.*,mytest\\..*,mytest2.user
canal.mq.partition=0
# hash partition config
#canal.mq.partitionsNum=3
#canal.mq.partitionHash=test.table:id^name,.*\\..*
#################################################
```
然后可以直接到canal解压后的bin目录中执行脚本启动:`sh startup.sh`

# Canal客户端接入
首先引入依赖：

```xml
<dependency>
    <groupId>com.alibaba.otter</groupId>
    <artifactId>canal.client</artifactId>
    <version>1.1.0</version>
</dependency>
```
直接复制官方的客户端示例代码：

```java
public class SimpleCanalClientExample {


public static void main(String args[]) {
    // 创建链接
    CanalConnector connector = CanalConnectors.newSingleConnector(new InetSocketAddress(AddressUtils.getHostIp(),
                                                                                        11111), "example", "", "");
    int batchSize = 1000;
    int emptyCount = 0;
    try {
        connector.connect();
        connector.subscribe();
        connector.rollback();
        int totalEmptyCount = 120;
        while (emptyCount < totalEmptyCount) {
            Message message = connector.getWithoutAck(batchSize); // 获取指定数量的数据
            long batchId = message.getId();
            int size = message.getEntries().size();
            if (batchId == -1 || size == 0) {
                emptyCount++;
                System.out.println("empty count : " + emptyCount);
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                }
            } else {
                emptyCount = 0;
                // System.out.printf("message[batchId=%s,size=%s] \n", batchId, size);
                printEntry(message.getEntries());
            }

            connector.ack(batchId); // 提交确认
            // connector.rollback(batchId); // 处理失败, 回滚数据
        }

        System.out.println("empty too many times, exit");
    } finally {
        connector.disconnect();
    }
}

private static void printEntry(List<Entry> entrys) {
    for (Entry entry : entrys) {
        if (entry.getEntryType() == EntryType.TRANSACTIONBEGIN || entry.getEntryType() == EntryType.TRANSACTIONEND) {
            continue;
        }

        RowChange rowChage = null;
        try {
            rowChage = RowChange.parseFrom(entry.getStoreValue());
        } catch (Exception e) {
            throw new RuntimeException("ERROR ## parser of eromanga-event has an error , data:" + entry.toString(),
                                       e);
        }

        EventType eventType = rowChage.getEventType();
        System.out.println(String.format("================&gt; binlog[%s:%s] , name[%s,%s] , eventType : %s",
                                         entry.getHeader().getLogfileName(), entry.getHeader().getLogfileOffset(),
                                         entry.getHeader().getSchemaName(), entry.getHeader().getTableName(),
                                         eventType));

        for (RowData rowData : rowChage.getRowDatasList()) {
            if (eventType == EventType.DELETE) {
                printColumn(rowData.getBeforeColumnsList());
            } else if (eventType == EventType.INSERT) {
                printColumn(rowData.getAfterColumnsList());
            } else {
                System.out.println("-------&gt; before");
                printColumn(rowData.getBeforeColumnsList());
                System.out.println("-------&gt; after");
                printColumn(rowData.getAfterColumnsList());
            }
        }
    }
}

private static void printColumn(List<Column> columns) {
    for (Column column : columns) {
        System.out.println(column.getName() + " : " + column.getValue() + "    update=" + column.getUpdated());
    }
}

}
```
控制台可以看到输出：

	empty count : 1
	empty count : 2
	empty count : 3
	empty count : 4

然后给数据库新增一条数据，即可在客户端看到数据变更和类型：

	================> binlog[master-bin.000045:237902.001946:313661577] , name[test,user] , eventType : INSERT
	ID : 4    update=true
	X : 2013-02-05 23:29:46    update=true

# 一些问题
假如出现rowChage.getRowDatasList()获得的数据集合为null或者集合中没有元素导致控制台并没有打印出以上信息（参照[issues#1267](https://github.com/alibaba/canal/issues/1267)以及[FAQ](https://github.com/alibaba/canal/wiki/FAQ)），通常情况下是由于两种原因引起的：

* MySql的配置文件里的binlog-format没有配置为ROW；
* 在canal的配置文件里，即example的instance.properties的配置规则过滤掉了数据，检查一下该规则；


# 总结
可以通过canal实现MySql到各种其他数据库的迁移。
