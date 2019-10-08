---
title: Spring MVC源码初探(二)-Servlet 3.0、SPI、Aware
categories: 学习笔记
date: 2019-10-07 17:40:48
tags: JAva
keywords: [SPI,Servlet]
description: Spring MVC学习笔记
---

# 简述

这篇笔记主要记录一些比较重要，但是上一篇又没有提到的内容。其中，Spring MVC对Servlet 3.0的实现是重中之重。

<!--more-->

# JAVA SPI

动态替换服务实现的机制，目前Dubbo就是基于SPI提供扩展机制。

## Demo

写一个小demo感受一下：

![](Spring-MVC源码初探-二-Servlet-3-0、SPI、Aware/Spi-Demo-Structure.png)