---
title: Docker入门学习二
categories: 学习笔记
date: 2019-02-25 14:31:52
tags:
- Docker
- CI/CD
keywords: [CI,CD,Docker]
description: 构建和编排自己的Docker应用
---

# Dockerfile

通过dockerfile实现秒级镜像迁移。

定义：

定义镜像自动化构建流程的配置文件。

组成：

注释行加指令行。

作用

* 快速迁移、部署；
* 记录了镜像构建顺序和逻辑；
* 自动化流程；
* 提交修改时只需要提交dockerfile的修改。

<!--more-->

## 结构

* 基础指令：定义新镜像的基础和性质；
* 控制指令：指导镜像构建的核心部分，用于描述镜像在构建过程中需要执行的指令；
* 引入指令：将外币文件引入到构建镜像内部；
* 执行指令：为容器指定在启动时需要执行的脚本或指令；
* 配置命令：配置网络、用户等。

## 常见指令

### FROM

一般情况下，我们不会从0开始搭建一个基础镜像，而是会选择一个已经存在的镜像为基础。

        FROM<image>[AS<name>]
        FROM<image>[:<tag>][AS<name>]
        FROM<image>[@<digest>][AS<name>]

也可以用该指令合并两个镜像的功能。

### RUN

用于向控台发送命令。

      RUN<command>
        RUN["executable","param1","param2"]

支持反斜杠换行。

### ENTRYPOINT/CMD

用于启动容器内的主程序。

    ENTRYPOINT["executable","param1","param2"]
    ENTRYPOINTcommandparam1param2
    CMD["executable","param1","param2"]
    CMD["param1","param2"]
    CMDdommandparam1param2

### EXPOSE

为容器暴露指定端口。

    EXPOSE<port>[<port>/<protocol>...]

两个命令大体相近，都可以为空。
当两个命令同时给出时，CMD的内容会作为ENTRYPOINT定义的命令的参数。

### VOLUME

定义数据卷目录

    VOLUME["/data"]

### COPY/ADD

从宿主机的文件系统里拷贝内容到镜像里的文件系统中。

    COPY[--chown=<user>:<group>]<src>...<dest>
    ADD[--chown=<user>:<group>]<src>...<dest>

src和dest外可以加""。
两者的区别在于：**ADD能够使用URL作为src，并且识别到源文件为压缩包时自动解压。**

`docker build <path>`path可以为本地路径或者URL路径，但并不是dockerfile文件路径，而是构建环境目录，-f可以指定dockerfile文件位置，未指定的话默认就在环境目录下去找，-t可以指定新生成镜像的名称。

### ARG

定义一个变量，变量只可以在build时传进来，只在当前镜像内有效。
例如dockerfile如下：

```
FROM debian:stretch-slim
...
ARG TOMCAT_MAJOR
ARG TOMCAT_VERSION
...
RUN wget -0 tomcat.tar.gz "http//...$TOMCAT_MAJOR:$TOMCAT_VERSION..."
...﻿​
```

在build时，可以这样传入参数：`docker build --build-arg TOMCAT_MAJOR=8 --build-arg TOMCAT_VERSION=8.0.53 ./`

### ENV

定义一个环境变量，环境变量在所有基于此镜像的镜像内都生效，并且可以指定值。

    FROM debian:stretch-slim
    ...
    ARG TOMCAT_MAJOR 8
    ARG TOMCAT_VERSION 8.0.53
    ...
    RUN wget -0 tomcat.tar.gz "http//...$TOMCAT_MAJOR:$TOMCAT_VERSION..."
    ...

取值与ARG一致，使用美元符号取值符，当ARG与ENV的名字重复时，ENV会覆盖ARG，同时ENV的值也可以通过运行时选项-e或者-env传入：
`docker run -e <key>=<value> <image>`。

## 指令实战

### 实战

构建一个Spring Boot（我的博客）项目，dockerfile如下：

```docker
FROM frolvlad/alpine-oraclejdk8:slim
VOLUME /tmp
COPY MyBlog-0.0.1-SNAPSHOT.jar app.jar
ENTRYPOINT ["java","-jar","-Dlogging.file=/spring.log","/app.jar"]
```

选择基础镜像含有oraclejdk8，如果不清楚，可以使用docker search oraclejdk8，复制其中一个的name作为基础镜像。同时，为容器挂载一个目录，主机的/var/lib/docker下会有一个目录挂载到/tmp目录下，根据Srping Boot官方说明：

> We added a VOLUME pointing to "/tmp" because that is where a Spring Boot application creates working directories for Tomcat by default. The effect is to create a temporary file on your host under "/var/lib/docker" and link it to the container under "/tmp". This step is optional for the simple app that we wrote here, but can be necessary for other Spring Boot applications if they need to actually write in the filesystem.

Spring Boot项目启动时是默认以/tmp目录作为tomcat的工作目录的，所以最好挂载一个宿主机到容器中，虽然对于这个简单项目这一步是可选的，但是对于其他Spring Boot项目可能是必须的。
使用COPY或者ADD命令拷贝程序到镜像文件系统中，然后最后一步是主要指令，需要记住：第一个是命令，每多一条参数，请多一个""（双引号），不然会报Unrecognized option: -jar -Dlogging.file=/spring.log（无法识别选项）的错。最后一步：docker build ./ -t myblog开始构建镜像，记住镜像名只能小写。

### 技巧

    #对于能够合并的多条指令，推荐合并。以下两种效果时一样的，但是推荐第一种
    RUN apt-get update;\
        apt-get istall -y --no-install-recommends $fetchDeps;\
        rm -rf /var/lib/apt/lists/*;
    
    RUN apt-get update;
    RUN apt-get install -y --no-install-recommends $fetchDeps;
    RUN rm -rf /var/lic/apt/lists/*

原因在于：每一条能够对文件系统的指令执行之前，Docker都会基于上条命令的结果启动一个容器，在容器中运行这条指令的内容，之后再形成一层镜像，如此反复形成最终镜像。因此，合并以后就能提升构建效率。

对于ENTRYPOINT和CMD，前者优先级高于后者，因为它常用于对容器初始化，而CMD才是启动主程序的指令，在前一篇基础知识中提过运行时重写指令，其实重写的就是CMD指令，当两者同时存在时，CMD指令会作为ENTRYPOINT指令的参数存在。

例如：

```docker
...
ENTRYPOINT ["sh","/usr/local/bin/git"]
CMD ["sh","/usr"]
## 以上两句等于：sh /usr/local/bin/git sh /usr
ENTRYPOINT /bin/ep arge
CMD /bin/exec args
##以上两句等于：/bin/sh -c /bin/ep arge /bin/sh -c /bin/exec args
...﻿​
```

注：两个指令如果不带中括号，那么默认执行方式是/bin/sh -c [中括号里参数]

可以观摩Docker Hub里更多大佬们写的Dockerfile。

同时Docker Hub也有各种官方镜像，例如Mysql如何在启动时传入root用户密码（初始化运行Mysql时是需要设置用户名密码的）。

我们也可以共享我们的镜像，前提是将Docker Hub账号授权到GitHub或者Bitbucket来从代码库中获取。

# Docker Compose

容器管理工具，也被称为容器编排工具。

一套项目，往往是由数据库，缓存，应用组成，而在分布式架构下，每一个实例可能都有多个，所以就需要容器编排工具进行管理。

Compose可以对本机上的容器进行编排。Kuberneters是由Google研发的可以对其他服务器上的容器进行管理的工具，Swarm是Docker公司研发的和K8s是类似的工具。

Docker Compose并不属于Docker Engine的一部分，所以需要另外安装。而对于windows或者mac，如果装了Docker Desktop（需要开启Hyper-v）或者Docker toolbox（需要安装virtualBox引擎）都可以直接运行Docker Compose（Hyper-v和virtualBox所使用的虚拟化技术会冲突，所以只推荐装其中一种）。

Docker Compose默认使用docker-compose.yml文件作为配置文件，如非必要不推荐改名。
看一个简单的配置文件：

```yaml
#yml已经更新到了第几版
version: '3'
#核心细节部分：配置应用集群
services:
  webapp:
    build: ./image/webapp
    ports:
      - "5000":"5000"
    volumes:
      - ./code:/code
      - logvolume: /var/log
    links:
      - mysql
      - redis
    redis:
      image: reids:3.2
    mysql:
      image: mysql:5.7
      enviroment
      - MYSQL_ROOT_PASSWORD=123
volumes:
  logvolume: {}
```

`docker-compose up`：运行，-d后台运行，-f修改指定配置文件，-p定义项目名称。
`docker-compose down`：停止所有容器，并将他们删除。
"随用随删，随用随启"

### 容器编排命令

    docker-compose logs <cointainer>：查看集群中某个容器内主进程日志
    docker-compose create/start/stop/ <cointainer> ：创建/启动/停止集群内某个容器

### 常用配置

```yaml
version: "3"
services:
    redis:
        ##直接指定镜像
        image: redis:3.2
        networks:
            - backend
            ## 指定驱动类型并指定子网网段
            # backend：
            # driver: bridge
            ## 指定别名
            # alias：
            #     - backend.redis
                ##ipam:
                    # driver: default
                    # config:
                        # - subnet: 10.10.1/24
        volumes:
            ##这里的相对目录是指相对docker-compose.yml所在目录
            - ./redis/redis.conf:/etc/redis.conf:ro
        prots:
            ## 需要注意的是，yml文件对于小于60的xx：xx格式会解析为时间，所以最好用双引号括起来
            - "6379":"6379"
        ##指定redis启动时的指令，此处是为了定义启启动的配置文件
        command: ["redis-server","/etc/redis.conf"]

    database:
        image: mysql:5.7
        networks:
            - backend
        volumes:
            - ./mysql/my.cnf:/etc/mysql/my.cnf:ro
            - mysql-data:/var/lib/mysql
        ##类似于-e或者-env传递参数
        environment:
            - MYSQL_ROOT_PASSWORD=my-secret-pw
        ##类似于-p指定端口映射
        ports:
            - "3306":"3306"

    webapp:
        ##没有指定镜像，说明镜像来源于构建
        build: ./webapp
        networks:  
            - frotend
            - backend
        volumes:
            - ./webapp:/webapp
        depends_on:
            - redis
            - database

    nginx:
        image: nginx:1.12
        networks:
            - frontend
        volumes:
            - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
            - ./nginx/conf.d:/etc/nginx/conf.d:ro
            - ./webapp/html:/webapp/html
        depends_on:
            - webapp
        ports:
            - "80":"80"
            - "443":"443"
networks:
    frontend;
    backend;
volumes:
    mysql-data
    ## 以下配置表示引用外部数据卷，即docker engine里已有的数据卷
    ##mysql-data:
        # exernal: true
```

### Docker Compose 实战

现有如下几个服务：

mysql，redis，tomcat，Java war包

建议性目录结构（以下都在项目目录下）：

* app：存放程序工程，即代码、编译结果以及相关的库、工具等；
* compose：定义docker compose项目；
* mysql：与mysql相关配置；
* redis：与redis相关配置
* tomcat：与tomcat相关配置

下面的配置都是根据官网修改我们所需而来的配置。mysql配置：

```
# ./mysql/my.cnf

[mysqld_safe]
pid-file = /var/run/mysqld/mysqld.pid
socket   = /var/run/mysqld/mysqld.sock
nice     = 0

[mysqld]
skip-host-cache
skip-name-resolve
explicit_defaults_for_timestamp

bind-address = 0.0.0.0
port         = 3306

user      = mysql
pid-file  = /var/run/mysqld/mysqld.pid
socket    = /var/run/mysqld/mysqld.sock
log-error = /var/log/mysql/error.log
basedir   = /usr
datadir   = /var/lib/mysql
tmpdir    = /tmp
sql_mode  = NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES

lc-messages-dir = /usr/share/mysql

symbolic-links = 0
```

redis配置：

```
# ./redis/redis.conf
##...
################################## SECURITY ###################################

# Require clients to issue AUTH <PASSWORD> before processing any other
# commands.  This might be useful in environments in which you do not trust
# others with access to the host running redis-server.
#
# This should stay commented out for backward compatibility and because most
# people do not need auth (e.g. they run their own servers).
#
# Warning: since Redis is pretty fast an outside user can try up to
# 150k passwords per second against a good box. This means that you should
# use a very strong password otherwise it will be very easy to break.
#
requirepass my-secret-pw
##...
```

也可以使用容器中默认的配置文件复制到宿主机以便我们在宿主机中进行操作。

```bash
#首先需要创建一个临时容器：
docker run --rm -d --name temp-tomcat tomcat:8.5
#然后使用docker cp命令复制容器内的tomcat配置文件到宿主机中：
docker cp temp-tomcat:/usr/local/tomcat/conf/server.xml ./server.xml
docker cp temp-tomcat:/usr/local/tomat/conf/web.xml ./web.xml
```

如果两个参数调换位置就是将宿主机中的配置文件复制到容器的文件系统里。

docker-compose.yaml:

```yaml
version: "3"

services:

  redis:
    image: redis:3.2
    volumes:
      ##使用宿主机的某个配置文件作为容器内程序配置文件，方便修改
      - ../redis/redis.conf:/etc/redis/redis.conf:ro
      ##将宿主机的某个目录挂载到容器内，保证数据不丢失
      - ../redis/data:/data
    command:
     ## 指定启动命令为 redis-server /etc/redis/redis.conf，多参数写多行
      - redis-server
      - /etc/redis/redis.conf
    ports:
     - 6379:6379

  mysql:
    image: mysql:5.7
    volumes:
      - ../mysql/my.cnf:/etc/mysql/my.cnf:ro
      ##将宿主机的某个目录挂载到容器内，保证数据不丢失
      - ../mysql/data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: my-secret-pw
    ports:
      - 3306:3306

  tomcat:
    image: tomcat:8.5
    volumes:
        ##将宿主机的项目目录下的app挂载到容器目录中
      - ../app:/usr/local/tomcat/webapps/ROOT
    ports:
      - 80:8080
```

# 服务化开发

## 搭建本地环境

使用docker compose定义本地的一组环境，如spring项目+mysql数据库。
与同事的其他服务模块产生调用（可以借助网络别名更轻松地调用）。
借助Overlay网络实现不同主机互联。

## Docker Swarm

Docker内置的集群工具，帮助管理部署不同主机上的Docker daemon集群。

    docker swarm init：初始化集群，默认当前节点为管理节点
    docker swarm join：加入集群，docker swarm join-token表示获得管理节点的加入命令，执行获得的那条管理节点的加入命令以后就可以称为管理节点
    docker network create：--driver overlay表示启动类型为跨网类型，--attachable mesh选项方便不同主机上的docker容器能够正常使用到它
    docker network ls：查看其下网络列表

然后在docker compose的yml文件中配置networks配置块：

```yaml
networks：
    mesh：
        external： true
```

# 服务发现

使用docker compose模拟zookeeper集群注册中心，用三个docker compose 服务定义这三个节点：

```yaml
version: '3'

services:

  zk1:
    image: zookeeper:3.4
    restart: always
    hostname: zk1
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=0.0.0.0:2888:3888 server.2=zk2:2888:3888 server.3=zk3:2888:3888
    ports:
      - 2181:2181

  zk2:
    image: zookeeper:3.4
    restart: always
    hostname: zk2
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zk1:2888:3888 server.2=0.0.0.0:2888:3888 server.3=zk3:2888:3888
    ports:
      - 2182:2181

  zk3:
    image: zookeeper:3.4
    restart: always
    hostname: zk3
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zk1:2888:3888 server.2=zk2:2888:3888 server.3=0.0.0.0:2888:3888
    ports:
      - 2183:2181
```

ZOO\_MY\_ID:zk在集群中的编号。

ZOO\_SERVICES:定义所有zk和它们的连接方式。

例如：

    server.1=0.0.0.0:2888:3888 server.2=zk2:2888:3888 server.3=zk3:2888:3888

我们可以在 ZOO_SERVERS 中定义所有处于 Zookeeper 集群中的程序，通过空格来间隔它们。而每个服务的的定义形式为 server.[id]=[host]:[port]:[port]，所以就有了上面例子中我们看到的样子。

在这个例子里，我们描述了三个 Zookeeper 程序的连接地址。

由于每个容器都有独立的端口表，所以即使这些程序都运行在一个主机里，我们依然不需要担心，它们会造成端口的冲突。所以这里我们直接使用默认的 2888 和 3888 来进行服务间的相互通信即可。

而在进行容器互联的过程中，我们可以通过 Docker 的解析机制，直接填入对应服务的名称替代它们的 IP 地址，也就是这个例子里的 zk2 和 zk3。

restart: always 这个配置，这个配置主要是用来控制容器的重启策略的。

这里的 always 指的是不论任何情况，容器出现问题后都会自动重启，也包括 Docker 服务本身在启动后容器也会自动启动。

| 配置值            | 说明                |
| -------------- | ----------------- |
| no             | 不设重启机制            |
| always         | 总是重启              |
| on-failure     | 在异常退出时重启          |
| unless-stopped | 除非由停止命令结束，其他情况都重启 |
