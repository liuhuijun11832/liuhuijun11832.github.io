---
title: Nexus简单搭建私服
categories: 编程技术
date: 2019-03-18 17:13:33
tags:
- Nexus
- Maven
- 随记
keywords: [Maven,Nexus] 
description: 记录使用Nexus搭建Maven私服的步骤
---
# 简述
Nexus['neksəs]是一个依赖管理的仓库，用于提供Maven和Gradle等构建工具的依赖源，它可以整合本地和远程的所有仓库并按照指定顺序寻找依赖，这里随手记载一下搭建过程。

# 下载安装
官网：[https://www.sonatype.com/download-oss-sonatype](https://www.sonatype.com/download-oss-sonatype)。
根据系统版本下载，本文使用的版本是3.10.0，Linux可以下载OS-X版本或者Unix版本，差别不大。
上传到服务器，执行以下命令，（假设现在具有root权限，解压到的目标目录可以随意选择）。

```bash
mkdir /home/nexus
tar -xzvf nexus-3.10.0-04-unix.tar.gz -C /home/nexus
useradd nexus
chown -R nexus /home/nexus
cd /home/nexus
./nexus-3.10-04/bin/nexus start
```
命令解释：nexus不推荐使用root执行启动脚本，所以这里我们自己新增了一个用户，并且修改了nexus目录的所属用户为nexus，当然，也可以不用修改所属用户，只需要给nexus用户增加一个写入权限即可：`chmod -R +w /home/nexus`，或者写入777权限`chmod -R 777 /home/nexus`。

# 仓库配置
