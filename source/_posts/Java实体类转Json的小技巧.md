---
categories: 编程技术
title: Java实体类转Json的小技巧
date: 2019-04-15 17:11:58
tags: 随记
keywords: [Json,Gson,FastJson]
description: Java使用fastjson和gson将实体类转化为json字符串的一些小技巧
---

# 简介

Google的Gson和Alibaba的FastJson都是常用的用于将实体类或者容器转化为json字符串的工具包，但是这两个工具包都有一个共同的小问题，默认情况下，对于bean里为null的字段，转为json以后会直接丢失不显示。但是有的时候为了方便前端，有时候想当String类型的字段为null时，返回给前端一个空串。

<!--more-->

## Gson

首先使用gson的方式来实现，我们需要写一个adapter继承gson的typeadapter来实现我们的转换逻辑。

```java
public class StringNullAdpater extends TypeAdapter<String> {

    @Override
    public String read(JsonReader reader) throws IOException {
        if (reader.peek() == JsonToken.NULL) {

            reader.nextNull();
            return "";
        }
        return reader.nextString();
    }
    @Override
    public void write(JsonWriter writer, String value) throws IOException {
        // 这里就是序列化为Json字符串的逻辑

        if (value == null) {
            writer.value("");
            return;
        }
        writer.value(value);
    }
}
```

## FastJson

fast的方式较为简单：

`JSON.toJSONString(pvLog, SerializerFeature.WriteNullStringAsEmpty,SerializerFeature.PrettyFormat)`，利用toJSONString的重载方法，通过添加SerializerFeature的枚举参数来实现，枚举具体类型部分参考如下：

    WriteMapNullValue,  //保留值为null的字段
    WriteEnumUsingToString,  //将枚举类型使用toString()方法输出
    WriteNullListAsEmpty,  //将空list输出为[]
    WriteNullStringAsEmpty,  //保留null String为输出为""
    WriteNullNumberAsZero,  //将null Number输出为0
    WriteNullBooleanAsFalse,  //将null Boolean输出为false
    SkipTransientField,  //不转换瞬态变量（即不参与序列化）
    SortField,  //排序字段
    WriteTabAsSpecial,  //将tab制表符做转义输出
    PrettyFormat,  //格式化json数据，更美观地查看
    WriteClassName,  //序列化时写入类信息 
    WriteDateUseDateFormat,//是否格式化时间

# 总结

对于spring mvc的项目，我们可以自定义配置Json转换器xxxHttpMessageConverter使得我们加了@ResponseBody的接口可以避免null字段丢失。
