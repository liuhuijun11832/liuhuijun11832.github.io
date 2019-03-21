---
title: Git+Hexo搭建个人博客
date: 2019-01-28 13:32:51
tags: 
- hexo
- 随记
categories: 编程技术
---

# 简述
环境：腾讯云服务器+新网域名，github，hexo

需求：[www.guitar-coder.cn](www.guitar-coder.cn)指向服务器主站，[blog.guitar-coder.cn](blog.guitar-coder.cn])指向github.io，写个人笔记和博客。
<!-- more -->

# 步骤
## 安装nodejs
下载nodeJS，官网https://nodejs.org/en/

windows或者mac可以下载可执行文件直接一路安装即可，当然mac也可以执行`brew install node`,至于linux下，这里以二进制binary为例（来源于官网教程）：
```bash
	#定义linux变量
	VERSION=v8.11.4
	DISTRO=linux-x64
	sudo mkdir /usr/local/lib/nodejs
	sudo tar -xJvf node-$VERSION-$DISTRO.tar.xz -C /usr/local/lib/nodejs 
	sudo mv /usr/local/lib/nodejs/node-$VERSION-$DISTRO /usr/local/lib/nodejs/node-$VERSION
	#修改~/.profile文件（用户变量）或者/etc/profile（全局环境变量）
	export NODEJS_HOME=/usr/local/lib/nodejs/node-$VERSION/bin
	export PATH=$NODEJS_HOME:$PATH
	#变量生效
	. ~/.profile
	node -v
	npm version
	#将node命令连接到系统命令目录下
	sudo ln -s /usr/local/lib/nodejs/node-$VERSION/bin/node /usr/bin/node
	
	sudo ln -s /usr/local/lib/nodejs/node-$VERSION/bin/npm /usr/bin/npm
	
	sudo ln -s /usr/local/lib/nodejs/node-$VERSION/bin/npx /usr/bin/npx```
为了方便访问，使用国内淘宝的npm镜像：

`npm install -g cnpm --registry=https://registry.npm.taobao.org﻿​`

以后npm和cnpm都可以用，效果基本一致，cnpm的速度更快。

## 安装hexo
首先需要安装hexo-cli脚手架工具：

`cnpm install hexo-cli -g`

然后创建一个空目录，名为blog，进入目录执行初始化：

`hexo init`

生成静态页面：

`hexo generate或者hexo g`

启动本地服务：

`hexo server或者hexo s`

出现如下结果，即可预览本地4000端口的个人主页，默认用的都是theme下的landscape主题：

	liuhuijundeMacBook-Air:blog liuhuijun$ hexo s
	INFO Start processing 
	INFO Hexo is running at http://localhost:4000 . Press Ctrl+C to stop.
如果遇到

	ERROR Local hexo not found in ~/Desktop/blog 
	ERROR Try running: 'npm install hexo --save'﻿
这种错，有可能是由于modules下载出现问题，删除node_modules文件夹，然后重新执行`cnpm install` 或者 `npm install`即可。


## 远程部署
在github或者gitee上新建一个仓库，可以不用gitignore，hexo已经自动生成，readme文件也可以随意。

在blog目录下找到.config.yml文件，编辑，找到deploy模块，改成仓库地址，记住yml文件格式要求value前面有一个空格。


	deploy:
   		type: git
   		repo: your url
   		branch: master

执行：`cnpm intall hexo-deployer-git --save`

每一次新写文章，可以通过`hexo clean && hexo generate && hexo deploy`发布到博客。


## 多机器开发
此时线上的仓库里会有hexo编译成js以后的文件，还需要新建分支存放源代码：

	git checkout -b source
	git add ./
	git commit -m ""
	git push origin source

master分支用于存放页面，source用于存放源代码。


## 自定义域名
将blog.guitar-coder.cn指向github提供的pages。默认情况下github提供的pages是Repository name，当然我的guitar-coder.cn已经备案过。

在github settings页面，填入自己的域名，如图：
![domain.png](git-hexo搭建个人博客/github pages custom domain.png)

然后在hexo的source文件夹内新建CNAME文件，里面只需要填入你的个人域名blog.guitar-coder.cn（无需http或者https）,记住一定要push到source分支并且deploy到master分支。

最后一定要配置好域名解析：
![tencent.png](git-hexo搭建个人博客/1548653104444_3.png)

因为我是要把blog.guitar-coder.cn转发到github pages的liuhuijun11832.github.io的页面，所以需要配置CNAME，这里也可以先ping 通liuhuijun11832.github.io的IP地址，然后使用记录类型A来进行解析。
每个DNS服务商的刷新速度不一致，静等片刻即可。

## 推送博客
新建一篇博文：
`hexo new "test"`

然后找一个能够实时渲染效果的markdown编辑器，mac下推荐macdown，windows下我习惯使用markdown pad2（需要安装awison-sdk插件实时渲染），打开source/_posts/test.md完成以后使用`hexo clean && hexo g&& hexo d`即可发布。

# 参考

[https://haoshuai6.github.io/2016-11-25-Hexo-github-cname.html](https://haoshuai6.github.io/2016-11-25-Hexo-github-cname.html)

个人博客演示地址：

[https://blog.guitar-coder.cn/](https://blog.guitar-coder.cn/)

支付宝/微信 二维码生成（app生成的太不美观）：
[https://cli.im/weixin](https://cli.im/weixin)

图标查询：
[https://fontawesome.com/icons?d=gallery](https://fontawesome.com/icons?d=gallery)



