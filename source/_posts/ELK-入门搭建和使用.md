---
title: ELK-入门搭建和使用
categories: 编程技术
date: 2019-06-21 14:19:03
tags:
- ELK
keywords: [ELK]
description: ELK的入门搭建和使用
---

# 简述

ELK是现在流行的数据采集、监控、分析系统，其本质是由三个开源组件ElasticSearch+Logstash+Kibana组成，常用于系统监控，日志分析等场景。

ElasticSearch:基于Json的分布式分析和搜索引擎，水平扩展，高可靠性，管理便捷，后面简称ES。

LogStash:动态数据收集管道，拥有可扩展的插件系统，强大的与ES的协同功能。

Kibana:以图表的形式呈现数据，具有可扩展的用户界面管理和配置Elastic Stack。

由于logstash比较重量级，所以有时候也会直接通过Beats输入到ES，但是更多的是beats+logstash的结合使用，即ES和Logstash放在同一台机器，用logstash做文本格式化，用轻量级的beat在端点采集日志传输到logstash。

<!--more-->

# 环境简述

- Linux Centos 7.2
- ElasticSearch 7.1.1
- Logstash 7.1.1
- FileBeat 7.1.1
- Kibina 7.1.1

日志文件类型：

- Java WEB：格式如下：

  ```prolog
  2019-04-02 08:47:40.067 [http-bio-9081-exec-4] DEBUG org.mybatis.spring.SqlSessionUtils [287]- Transaction synchronization deregistering SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@5ce37a0d]
  2019-04-02 08:47:40.067 [http-bio-9081-exec-4] DEBUG org.mybatis.spring.SqlSessionUtils [292]- Transaction synchronization closing SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@5ce37a0d]
  2019-04-02 08:47:40.070 [http-bio-9081-exec-4] DEBUG druid.sql.Connection [129]- {conn-10008} commited
  2019-04-02 08:47:40.070 [http-bio-9081-exec-4] DEBUG druid.sql.Connection [129]- {conn-10008} setAutoCommit true
  2019-04-02 08:47:40.071 [http-bio-9081-exec-4] DEBUG druid.sql.Connection [129]- {conn-10008} pool-recycle
  2019-04-02 08:47:43.034 [http-bio-9081-exec-10] INFO  c.a.ykp.filter.RedisSessionFilter [47]- RequestURI----->/fp/list.do
  2019-04-02 08:47:43.047 [http-bio-9081-exec-10] INFO  c.a.ykp.service.impl.BaseServiceImpl [188]- 接口编码为ECREST.DDLB.CX.E_INV的请求信息为>>>>>>>>{"NSRSBH":"140301201811057","FJH":"0","KPLX":"","DDSJQ":"","DDSJZ":"","GHFMC":"","GHF_EMAIL":"","GHF_SJ":"","FPKJ_ZT":"","PAGE_START":"0","PAGE_COUNT":"10","CZLX":"0","FPZL_DM":""}
  ```

- APP：格式如下：

```prolog
44    20190614    184715    0    07    ------    1019MakeInvCrypt  ret= 0
45    20190614    184715    0    07    ------    1019 MakeInv_Proc_2 结束
46    20190614    184715    0    07    ------    1019
47    20190614    184715    0    07    ------    c
1400171320V1B13.00H
12782637V0.03,100.00,3.00;
20190614 18:47:16
购货方名称
140301000000000
购方地址、邮编

661850000021
*园艺产品*石头￠§hwq14184713752IqiCUU@PTZCCS01
1
C180企业三十
110101000000030
杏石口路甲18号1# 020-28381999---1
111111111111111111111111111--1
开票人
复核人
收款人
wLbGsbXEt6LGsbG416I=
100.00
3.00
N
*园艺产品*石头
```

- 某硬件打印日志：

```prolog
963    2019-07-03    05:47:21    2A    0    超时触发
964    2019-07-03    05:47:21    2A    0    夜间空闲，开始自动扫描
965    2019-07-03    05:47:21    2A    0    scan_flag的值，0夜间自动 -1，业务错误，不会执行扫描，1，2手动触发
966    2019-07-03    05:47:21    2A    0    启动中触发自动扫描，不进行批量扫描 scan_flag == 0 && scanStartFlg == 1
967    20190703    055128    0    31    ------    DDZQ2019-07-03 05:51:28 172
968    20190703    055128    0    31    ------    interfaceCodeECXML.DDZQ.BC.E_INV.V2
969    20190703    055128    0    31    ------    m_WSAddrhttp://192.168.15.107:18082/51ykp/eInvoice
```

# 安装ES

详情请参考原来我写的一篇Skywaking的文章：[skywalking基础实践](https://blog.guitar-coder.cn/SkyWalking%E5%9F%BA%E7%A1%80%E5%AE%9E%E8%B7%B5.html)。

# 安装LogStash

## 启动前配置
需要注意的配置项：

ElasticSearch的配置文件config/elasticsearch.yml的`network.host`如果没有配置为"0.0.0.0"，那么就只能允许指定的IP访问，所以要保证logstash的配置文件里的IP为es指定的IP。

`192.168.29.30`上的ES配置的`network.host`为`192.168.29.30`，所以这里修改的logstash配置文件config/logstash-sample.conf为：
```
output {
  elasticsearch {
    hosts => ["http://192.168.29.30:9200"]
    index => "%{type}-%{+YYYY.MM.dd}"
    #user => "elastic"
    #password => "changeme"
  }
```
如果es配置了用户名和密码，那么也要进行对应的配置。
## 启动方式
启动脚本：

1. 进入到logstash所在目录`cd /elk/logstash-7.1.1`;
2. 启动方式（保证当前用户具有该目录下的可执行权限）：`nohup ./bin/logstash -f config/logstash-sample.conf >/dev/null 2>&1 &`;
3. 查看日志`tail -f logs/logstash-plain.log`，如果提示


	    [2019-06-19T16:38:00,235][INFO ][org.logstash.beats.Server] Starting server on port: 5044
	    [2019-06-19T16:38:00,386][INFO ][filewatch.observingtail  ] START, creating Discoverer, Watch with file and sincedb collections
	    [2019-06-19T16:38:00,769][INFO ][logstash.agent           ] Successfully started Logstash API endpoint {:port=>9600}

即代表启动成功。

## 完整配置说明
配置文件主要由三大部分组成，input可以配置数据来源：

- `type`：自定义，这里我配置为项目的名称；
- `path`：指定了收集的日志所在目录；
- `codec`：是一项通用配置，不光可以用于file input，其中：
    - `charset`：指定了读取的文件的编码格式，如果不配置charset，默认是`UTF-8`，
    - `pattern`：匹配该正则的日志会被收集，可以使用grok表达式；
    - `negate`：当日志不匹配时是否转置正则；
    - `what`：把当前行追加到前一行后面，直到新进行匹配pattern;
- `start_position`指定了日志的读取起始位置；
- `add_field`：添加新成员到收集的数据中，即在收集过来的json中添加
```
"fields": {
      "app_id": "app-to51"
    }
```

filter配置数据的格式：
根据自定义的成员fields里的app_id的值，判断来源于哪一个系统，根据不同系统日志的格式，使用grok表达式进行格式化输出，收集过来的日志数据都放在message中，所以这里格式化message：


	例如：从web收集过来的日志如下：
	2019-06-20 10:15:51.389 [http-bio-9081-exec-5-110101100000031-gxSplb262] INFO  c.a.ykp.service.impl.BaseServiceImpl [188]- 接口编码为ECREST.SPXXTB.CX.E_INV的请求信息为iNSRSBH\":\"110101100000031\",\"USER_ID\":\"12677\",\"ZHGXSJ\":\"2019-06-19 17:32:45\"}
	grok规则如下：
	match => {"message" => "%{TIMESTAMP_ISO8601:51web-date}%{SPACE}\[%{DATA}\]%{SPACE}%{WORD:51web-level}%{SPACE}%{DATA:51web-class}%{SPACE}\[\d+\]-%{SPACE}%{DATA:51web-desc}>+(?<web-data>.*)"}
	那么输出后存放到es中的json就会添加对应的字段，类似下面：
	{
	    ...
	    "web-data": "{\"NSRSBH\":\"911403016NLWYA33EE\",\"FJH\":\"0\",\"KPLX\":\"\",\"DDSJQ\":\"\",\"DDSJZ\":\"\",\"GHFMC\":\"\",\"GHF_EMAIL\":\"\",\"GHF_SJ\":\"\",\"FPKJ_ZT\":\"\",\"PAGE_START\":\"0\",\"PAGE_COUNT\":\"10\",\"CZLX\":\"0\",\"FPZL_DM\":\"\",\"DDSJBZ\":\"0\"}",
	    "51web-level": "INFO",
	    "51web-date": "2019-06-21 11:39:06.382",
	    "51web-class": "c.a.ykp.service.impl.BaseServiceImpl",
	    "51web-desc": "接口编码为ECREST.DDLB.CX.E_INV的请求信息为",
	    ...
	}


output配置了输出目的，这里配置输出到ES，还可以配置输出到控制台，方便调试。



	input {
	  beats {
	    port => 5044
	  }
	  file  {
	    type => "app-to51"
	    path => ["/elk/app/*","/elk/to51/*"]
	    codec => multiline {
	        charset => "GBK"
	        pattern => ^(\d+)(\s|\t)+(\d{4}-?\d{2}-?\d{2})(.|\n)*
	        negate => true
	        what => "previous"
	    }
	    start_position => "beginning"
	    add_field => {
	        "[fields][app_id]" => "app-to51"
	    }
	  }
	}
	
	filter {
	    if [fields][app_id] == "51-web"{
	        grok{
	            match => {"message" => "%{TIMESTAMP_ISO8601:51web-date}%{SPACE}\[%{DATA}\]%{SPACE}%{WORD:51web-level}%{SPACE}%{DATA:51web-class}%{SPACE}\[\d+\]-%{SPACE}%{DATA:51web-desc}>+(?<web-data>.*)"}
	            
	            remove_field => {"message"}
	        }
	    }else if  [fields][app_id] == "app-to51"{
	    grok {
	        match => {"message" => "\d+%{SPACE}(?<app-to51.date>\d{4}-?\d{2}-?\d{2}%{SPACE}\d{2}:?\d{2}:?\d{2})%{SPACE}(?<app-to51.action>\w+)%{SPACE}(?<app-to51.code>\d+)%{SPACE}(?<app-to51.message>(.|\n|\t)*)"}
	        remove_field => ["message"]
	    }
	  }
	}


为了方便启动，我在logstash目录下编写了一个start.sh脚本，方便启动，stop.sh脚本方便关闭。
## grok表达式
grok表达式内置了一些变量，这些变量能直接引用作为正则表达式，logstash启动的时候会将这些变量替换为正则，详情参考[https://github.com/logstash-plugins/logstash-patterns-core/blob/master/patterns/grok-patterns](https://github.com/logstash-plugins/logstash-patterns-core/blob/master/patterns/grok-patterns)：


	语法一：
		%{TIMESTAMP_ISO8601:51web-date}
		符合TIMESTAMP_ISO8601格式的日志文本会作为字段51web-date的内容输出到es里，TIMESTAMP_ISO8601就是grok预置变量；
	语法二：
		(?<app-to51.date>\d{4}-?\d{2}-?\d{2}%{SPACE}\d{2}:?\d{2}:?\d{2})
		这句的含义就是形式20190621 171600或者2019-06-21 17：16：00的文本作为app-to51.date字段的内容输出到es里。
		grok是基于正则表达式，所以也可以直接用正则表达式来匹配从而输出自定义字段内容，语法为(?<自定义字段名>正则表达式)。
	
# filebeat配置
配置文件为filebeat目录下的filebeat.yml，yaml配置的**配置名称和值中间有一个空格**。
## inputs配置
- `- type：log`：filebeat会按行读取log类型的文件；
- `enable：true`：启用该配置；
- `path: - /home/tomcat9080/logs/catalina.out`：收集的日志目录，可以使用*进行模糊匹配，也可以配置多个路径；
- `document_type`：写入es的文档类型；
- `tags`：标签，可以无需配置；
- `include_lines`：正则，采集匹配该正则的日志；
- `multiline.pattern`：多行日志的日志起始标志；
- `multiline.negate`：改为true，启用转置multiline.pattern（具体作用不明确）；
- `multiline.match`：改为after，表示合并起始行后面的行；
- `fields.app_id`：自定义的字段，会在发送到logstash的json数据中添加：
```
"fields": {
      "app_id": "app-to51"
    }
```

## output配置
由于没有将filebeat直接输出到es所以这里将所有`Elasticsearch output`的配置全部注释掉;

只需要Logstash output：


	#----------------------------- Logstash output --------------------------------
	output.logstash:
	  # The Logstash hosts
	  hosts: ["192.168.29.30:5044"]

## 启动
进入到filebeat目录下执行以下命令：
`nohup ./filebeat >/dev/null 2>&1 &`
为了方便启动，我在filebeat目录下编写了一个start.sh脚本，方便启动，stop.sh脚本方便关闭。

