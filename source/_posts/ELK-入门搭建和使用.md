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

```
 beats {
    port => 5044
  }
  file  {
    type => "hezi-app"
    path => ["/elk/app/*"]
    codec => multiline {
        charset => "GBK"
        pattern => "^(\d+)(\s|\t)+(\d{4}-?\d{2}-?\d{2})(.|\n)*"
        negate => true
        what => "previous"
    }
    start_position => "beginning"
    add_field => {
      "[fields][app_id]" => "hezi-app"
    }
  }
 file  {
    type => "hezi-to51"
    path => ["/elk/to51/*"]
    codec => multiline {
        charset => "GBK"
        pattern => "^(\d+)(\s|\t)+(\d{4}-?\d{2}-?\d{2})(.|\n)*"
        negate => true
        what => "previous"
    }
    start_position => "beginning"
    add_field => {
      "[fields][app_id]" => "hezi-to51"
    }
  }

}

filter {
    if [fields][app_id] == "51-web"{
    grok {
        match => {"message" => "%{TIMESTAMP_ISO8601:51web-date}%{SPACE}\[%{DATA}\]%{SPACE}%{WORD:51web-level}%{SPACE}%{DATA:51web-class}%{SPACE}\[\d+\]-%{SPACE}%{DATA:51web-desc}>+(?<web-data>.*)"}
        remove_field => ["message"]
    }
    grok {
        match => {"message" => "%{TIMESTAMP_ISO8601:51web-date}%{SPACE}\[%{DATA}\]%{SPACE}%{WORD:51web-level}%{SPACE}%{DATA:51web-class}%{SPACE}\[\d+\]-%{SPACE}(?<web-data>.*)"}
        remove_field => ["message"]
    }
  }else if [fields][app_id] == "hezi-app"{
    grok {
        match => {"message" => "\d+%{SPACE}(?<app-to51.date>\d{4}-?\d{2}-?\d{2}%{SPACE}\d{2}:?\d{2}:?\d{2})%{SPACE}(?<app-to51.action>\w+)%{SPACE}(?<app-to51.code>\d+)%{SPACE}(?<app-to51.message>(.|\n|\t|\s)*)"}
        remove_field => ["message"]
    }
  }
  else if [fields][app_id] == "hezi-to51"{
    grok {
        match => {"message" => "\d+%{SPACE}(?<app-to51.date>\d{4}-?\d{2}-?\d{2}%{SPACE}\d{2}:?\d{2}:?\d{2})%{SPACE}(?<app-to51.action>\w+)%{SPACE}(?<app-to51.code>\d+)%{SPACE}(?<app-to51.message>(.|\n|\t|\s)*)"}
        remove_field => ["message"]
    }
   grok {
        match => {"message" => "\d+%{SPACE}(?<app-to51.date>\d{4}-?\d{2}-?\d{2}%{SPACE}\d{2}:?\d{2}:?\d{2})%{SPACE}(?<app-to51.code>\d*)%{SPACE}(?<app-to51.action>\w+)%{SPACE}(?<app-to51.message>(.|\n|\t|\s)*)"}
        remove_field => ["message"]
    }

  }

}

output {
  elasticsearch {
    hosts => ["http://192.168.29.30:9200"]
    index => "%{[fields][app_id]}-%{+YYYY.MM.dd}"
    #user => "elastic"
    #password => "changeme"
  }
  #stdout { codec => rubydebug }
}
```
