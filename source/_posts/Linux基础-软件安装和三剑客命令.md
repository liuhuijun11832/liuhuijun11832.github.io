---
title: Linux基础-软件安装和三剑客命令
categories: 学习笔记
date: 2019-03-11 10:49:13
tags:
- Linux
- Vim
keywords: [Linux,Vim]
description: Linux的软件安装和grep、sec、awk命令等
---
# 简述
主要记录一些进阶内容。例如文本搜索命令grep，文本处理命令sed/awk，软件安装卸载，vim的高级配置等。
<!--more-->
 
# 软件卸载及安装
    RPM：RedHat Package Manager，红帽软件包管理器，通过一套本地数据库提供了一种更简单的软件安装管理方式。总共有两种类型，预编译包和源码包。预编译包只要在特定kernel版本下预先编译好程序（使用大多数常用编译参数），并将所需要的文件整体打包，在新主机上安装时，将文件直接复制到特定目录下即可；而二进制包需要自定义编译参数，需要先执行congfigure指令。
    
常用参数：

```bash
#安装软件
-i,--install
#打印详细信息
-v,--verbose
#打印安装过程（需要和-v一起用）
-h,--hash
#卸载软件
-e,--erase
#升级软件
-U,--upgrade=<packagefile>+
如果软件已经安装，则强行安装
--replacepkge
#安装测试，并不实际安装
--test
#忽略软件包依赖关系强行安装
--nodeps
#忽略软件包以及文件的冲突
--force
#查询参数（需要使用-q或者--query参数）
#查询所有安装软件
-a,--all
#查询某个安装软件
-p,--package
#列出某个软件包所包含的所有文件
-l,--list
#查询某个文件的所属包
-f,--file﻿​
```

yum：Yellow dog Updater Modified，一个基于RPM的shell前端包管理器，能够从指定服务器上自动下载并安装或者更新软件、删除软件，最大的好处是自动解决依赖关系，RedHat和CentOS的版本为5以上的都会默认安装yum。

```bash
yum [options] [command] [package]
#安装某个包
yum install PACKAGE
#安装某个软件组
yum groupinstall GROUP
#更新某个包
yum update PACKAGE
#更新某个软件组
yum groupupdate GROUP
#检查当前系统中需要更新的包
yum check-update
#显示软件源中所有可用的包
yum list
#显示系统已安装的包
yum list installed
#显示某个包的信息
yum info PACKAGE
#显示某个软件组的信息
yum groupinfo GROUP
#显示软件源宏所有的可用软件组
yum grouplist
#删除某个包
yum remove PACKAGE
#删除某个软件组
yum groupremove GROUP
#清除yum所生成的缓存文件
yum clean ﻿​
```

默认情况下RedHat会因为未注册RHN而无法使用yum，删除/etc/yum.repos.d/rhel-debuginfo.repo，创建一个新的Centos-Base.repo文件，输入以下内容，注意，不同版本的内容可能不一样：

```properties
[base]
name=CentOS-$releasever
enabled=1
failovermethod=priority
baseurl=http://mirrors.cloud.aliyuncs.com/centos/$releasever/os/$basearch/
gpgcheck=1
gpgkey=http://mirrors.cloud.aliyuncs.com/centos/RPM-GPG-KEY-CentOS-7
[updates]
name=CentOS-$releasever
enabled=1
failovermethod=priority
baseurl=http://mirrors.cloud.aliyuncs.com/centos/$releasever/updates/$basearch/
gpgcheck=1
gpgkey=http://mirrors.cloud.aliyuncs.com/centos/RPM-GPG-KEY-CentOS-7
[extras]
name=CentOS-$releasever
enabled=1
failovermethod=priority
baseurl=http://mirrors.cloud.aliyuncs.com/centos/$releasever/extras/$basearch/
gpgcheck=1
gpgkey=http://mirrors.cloud.aliyuncs.com/centos/RPM-GPG-KEY-CentOS-7﻿​
```

如果通过更改以上更改软件源的方式还是不行，那么需要使用rpm卸载原来的yum工具，然后重新安装CentOS的yum工具：`rpm -ivh rpm-url \ rpm-url\`。

# 文本搜索命令grep
grep：Global search Regular Expression and Print out the line。Linux下非常强大的基于行的正则文本搜索工具，区分大小写。

常用参数：

```bash
#不区分大小写
-i
#统计包含匹配行的行数
-c
#输出行号
-n
#反向匹配
-v﻿
```

常用正则：

* .    匹配换行符以外的任意一个字符
* \*    匹配前一个字符0次或任意多次
* \\{n,m\\}    匹配字符的重复次数至少n次，最多m次
* \\{n,\\}    匹配字符的重复次数至少n次
* \\{n\\}    匹配字符重复次数为n次
* ^    匹配开头字符，出现在方括号中即是取反的意思
* $    匹配结束字符
* []    匹配方括号内出现的任意字符
* \    转义字符，用于将某语义转义为其他语义，例如^代表匹配开头，那么\^表示匹配^字符
* \\<\\>    匹配左右边界
* \d    匹配一个数字
* \b    匹配单词边界
* \B    匹配非单词边界
* \w    匹配数字、字母、下划线。等价于[0-9A-Za-z]
* \W    匹配非字母、非数字、非下划线
* \n    匹配一个换行符
* \r    匹配一个回车符
* \t    匹配一个制表符
* \f    匹配一个换页符
* \s    匹配任何空白字符
* \S    匹配任何非空白字符

扩展正则表达（需要使用egrep命令）：

* ?    匹配前一个字符0或者1次
* \+    匹配前一个字符1次或以上
* |    或
* ()    配合|，（A|B|C），枚举一系列可能匹配的字符
* {}    匹配所有大括号内包含的以都好隔开的字符
* !    常和[]一起用，取反


常用命令：

```bash
grep '^good' test.txt   #匹配test.txt文件中以good开头的行
grep 'good$' test.txt   #匹配以good结尾的行  
grep  -c '^$' test.txt  #统计空行的数量
grep '[Gg]ood' test.txt #匹配good或者Good的行
grep '[^Gg]ood' test.txt    #匹配非Good或非good的行
grep 'g..d' test.txt    #匹配g后面有两个字符，第四个为d的行
grep '[Gg]..d' test.txt #匹配第一个字符为G或者g，第二、三位任意字符，第四个位d的行
grep '[Gg][lo].d' test.txt  
grep 'good' test.txt    #精确匹配包含这4个字符的行
grep '\<gold\>' test.txt    #匹配包含gold单词的行
grep '\bgold\b' test.txt    #匹配包含gold单词的行
grep go*d test.txt      #匹配以g开头，后面包含0或多个o，再跟一个d的单词的行
grep 'g.*d' test.txt    #匹配以g开头，后面有0或多个其他字符，最后一个是d单词的行
grep 'gl[0-9]' test.txt #匹配以gl开头，后面紧跟任意数字的单词的行
grep 'www\.baidu\.com' test.txt #匹配包含www.baidu.com的行
grep 'go\{2,\}' test.txt    #匹配g开头，至少包含2个o的单词的行
grep 'go\{4\}' test.txt     #匹配g开头，一定包含4个o的单词的行
#[:alnum:]  文字数字字符
#[:alpha:]  文字字符
#[:digit:]  数字字符
#[:graph:]  非空字符（非空格、控制字符）
#[:lower:]  小写字符
#[:cntrl:]  控制字符
#[:print:]  非空字符（包括空格）
#[:punct:]  标点符号
#[:space:]  所有空白字符（新行，空格，制表符）
#[:upper:]  大写字符
#[:xdigit:] 十六进制数字
grep ^[[:upper:]] test.txt  #匹配以大写字母开头的行
grep ^[[:digit:]] test.txt﻿​#匹配数字字符开头的行
ps -ef | grep PName   #将ps -ef搜寻到的进程交由grep匹配包含PName的进程
```

# 文本处理工具sed
sed：stream editor是一种非交互式的流编辑器，通过多种转换修改流经它的文本。默认情况下，sed并不会改变源文件本身，而只是对流经sed命令的文本进行修改，并将修改后的结果打印到标准输出中，但是-i选项会修改源文件。

使用场景：

* 常规编辑器编辑困难的文本。
* 过于庞大的文本，比如vi一个几百兆的文件。
* 有规律的文本修改，加快文本处理速度，比如全文替换。


用法：
`sed [options] 'command' file`

	#options是set可以接收的参数
	#command是sed的命令集（一共有25个）
	#使用-e参数和分号链接多编辑命令
	#-e的作用是将下一个字符串解析为sed编辑命令
	#将全局的this替换为That，将全局的line替换为LINE:sed -e 's/this/That/g' -e 's/line/LINE/g' 1.txt 
	#上个命令等同于：sed 's/this/That/g ; s/line/LINE/g' 1.txt
	
## 删除

	
```bash
#删除第一行
sed '1d' 1.txt
#将删除了第一行的流输入到2.txt里
sed '1d' 1.txt >> 2.txt
#删除从第1行开始的3行
sed '1,3d' 1.txt
#删除第一行到最后一行（清空）
sed '1,$d' 1.txt
#删除最后一行
sed '$d' 1.txt
#删除第五行以外的行
sed '5!d' 1.txt
#删除包含Empty的行
sed '/Empty/d' 1.txt
#删除空行
sed '/^$/d' 1.txt
```

## 替换
```bash
#默认情况下只替换第一次匹配到的内容
sed 's/line/LINE/' 1.txt
#最后一个斜杠后为替换范围，g表示全部
#替换两次的line为LINE
sed 's/line/LINE/2' 1.txt
#替换全部的line为LINE
sed 's/line/LINE/g' 1.txt 
#只替换开头的this为that
sed 's/^this/that/' 1.txt
```

## 字符转换
```bash
#将file中的O转换为N，L转换为E，D转换为W，转换字符和被转换字符长度要相等，否则无法执行
sed 'y/OLD/NEW/' file
```

## 插入

```bash
#在第二行之前插入Insert
sed '2 i Insert' 1.txt
#在第二行之前插入Insert
sed '2 a Insert' 1.txt
#在匹配行的上一行插入Insert
sed  '/Second/i\Insert' 1.txt
```

## 读文件

```bash
#将/etc/passwd中的内容读出放到1.txt空行之后
sed '/^$/r /etc/passwd' 1.txt
#使用p命令可以进行打印，这里使用sed命令时一定要加-n参数，表示不打印没关系的行
#打印出文件中指定的行
sed -n '1p' 1.txt
#将the替换为THE，sed实际处理了第二行，其他几行由于没有匹配所以并未真正处理，但是sed是基于流的，所以所有流过的行都打印出来了
sed 's/the/THE/' 1.txt
#可以使用-n只输出处理过的行
sed -n  's/the/THE/p' 1.txt
```
## 写文件
```bash 
#将1.txt文件里的第1，2行保存到output文件里
sed -n '1,2 w output' 1.txt
```
## sed脚本
新建一个sed.rules文件，内容如下：

```bash
#替换所有this为that
s/this/THAT
#删除空行
/^$/d﻿​
```

使用-f参数指定脚本应用于1.txt:
`sed -f sed.rules 1.txt`

## 高级替换

几种空间的概念：

模式空间：当前输入行的缓冲区；

缓存空间（存储空间）：sed处理完的流的缓冲区。

* H：将模式空间的内容追加到缓存空间
* h：将模式空间的内容复制到存储空间，覆盖原有存储空间
* G：将存储空间的内容追加到模式空间
* g：将存储空间的内容复制到模式空间

```bash
#读取匹配行的下一行（命令n代表下一行，参数-n代表处理过的行），再使用n命令后的编辑指令来处理该行
sed '/^$/{n;s/line/LINE/g}' 1.txt
#实现匹配a的行与匹配b的行的互换
sed '/a/{h;d};/b/G' Sed.txt 
```


sed常用命令：

|sed命令	|作用|
|---|---|
|a|	在匹配行后面加入文本|
|c|	字符转换|
|d	|删除行|
|D	|删除第一行|
|i	|在匹配行之前加入文本|
|h	|复制模式空间内容到存储空间|
|H	|将模式空间的内容追加到缓存空间|
|g	|将存储空间的内容复制到模式空间|
|G	|将存储空间的内容追加到模式空间|
|n	|读取下一个输入行，用下一个命令处理新的行|
|N	|追加下一个输入行到模式空间并在二者间插入新行|
|p	|打印匹配的行|
|P  |打印匹配的第一行|
|q	|退出sed|
|r	|从外部文件中读取文本|
|w   |	追加读写文件|
|！	|匹配取反|
|s/old/new	|用new替换正则表达式old|
|=	|打印当前行号|

sed常用参数：


|sed参数|	作用|
|---|---|
|-e|	多条件编辑|
|-h	|帮助信息|
|-n	|不输出不匹配的行|
|-f	|指定sed脚本|
|-V	|版本信息|
|-i	|修改源文件|

# 文本处理工具awk
awk是基于列的文本处理工具，能够处理结构化的文件，也就是说都是由各种单词和空白字符组成的，这里的空白字符包括空格，制表符，以及连续的空格和制表符等。每个非空白的部分叫做域。$1表示第一个域，$0表示全部域。

格式以及说明：


    awk [-F|-f|-v] ‘BEGIN{} //{command1; command2} END{}’ file
     [-F|-f|-v]   大参数，-F指定分隔符，-f调用脚本，-v定义变量 var=value
    '  '          引用代码块
    BEGIN   初始化代码块，在对每一行进行处理之前，初始化代码，主要是引用全局变量，设置FS分隔符
    //           匹配代码块，可以是字符串或正则表达式
    {}           命令代码块，包含一条或多条命令
    ；          多条命令使用分号分隔
    END         结尾代码块，在对每一行进行处理之后再执行的代码块，主要是进行最终计算或输出结尾摘要信息
	$0           表示整个当前行
    $1           每行第一个字段
    NF          字段数量变量
    NR          每行的记录号，多文件记录递增
    FNR        与NR类似，不过多文件记录不递增，每个文件都从1开始
    \t            制表符
    \n           换行符
    FS          BEGIN时定义分隔符
    RS      	 输入的记录分隔符， 默认为换行符(即文本是按一行一行输入)
    ~            匹配，与==相比不是精确比较
    !~           不匹配，不精确比较
    ==         等于，必须全部相等，精确比较
    !=           不等于，精确比较
    &&　     逻辑与
    ||             逻辑或
    +            匹配时表示1个或1个以上
    /[0-9][0-9]+/   两个或两个以上数字
    /[0-9][0-9]*/    一个或一个以上数字
    FILENAME 文件名
    OFS      输出字段分隔符， 默认也是空格，可以改为制表符等
    ORS        输出的记录分隔符，默认为换行符,即处理结果也是一行一行输出到屏幕
    -F'[:#/]'   定义三个分隔符
    
常用命令和功能：

```bash
打印指定域：
awk '{print $1,$4}'    Awk.txt
等同于：cat Awk.txt | awk '{print $1,$4}'
以下类似。
打印全部域：
awk '{print $0}'    Awk.txt
默认情况下以空白字符作为分隔符，可以指定字符为分隔符：
指定"."作为分隔符：
awk -F . Awk.txt
使用内置变量NF统计每行的列数：
awk '{print NF}' Awk.txt
$NF表示最后一列：
awk '{print $NF}' Awk.txt
$(NF-1)代表倒数第二列：
awk '{print $(NF-1)}' Awk.txt
substr函数能截取指定域，substr(域，begin，end)：
打印第一个域的第六个字符到最后一个字符：
cat Awk.txt | awk '{print substr($1,6)}'
打印每一行长度：
cat Awk.txt | awk '{print length}'
求列和(第三列年龄和)：
cat Awk.txt | awk 'BEGIN{total=0}{total+=$3}{print total}'
求平均年龄：
cat Awk.txt | awk 'BEGIN{total=0}{total+=$3}{print total/NR}'
```
# vim配置优化

```vim
if v:lang =~ "utf8$" || v:lang =~ "UTF-8$"
   set fileencodings=ucs-bom,utf-8,latin1
endif

set nocompatible        " Use Vim defaults (much better!)
set bs=indent,eol,start         " allow backspacing over everything in insert mode
"set ai                 " always set autoindenting on
"set backup             " keep a backup file
set viminfo='20,\"50    " read/write a .viminfo file, don't store more
                        " than 50 lines of registers
set history=50          " keep 50 lines of command line history
set ruler               " show the cursor position all the time

" Only do this part when compiled with support for autocommands
if has("autocmd")
  augroup redhat
  autocmd!
  " In text files, always limit the width of text to 78 characters
  " autocmd BufRead *.txt set tw=78
  " When editing a file, always jump to the last cursor position
  autocmd BufReadPost *
  \ if line("'\"") > 0 && line ("'\"") <= line("$") |
  \   exe "normal! g'\"" |
  \ endif
  " don't write swapfile on most commonly used directories for NFS mounts or USB sticks
  autocmd BufNewFile,BufReadPre /media/*,/run/media/*,/mnt/* set directory=~/tmp,/var/tmp,/tmp
  " start with spec file template
  autocmd BufNewFile *.spec 0r /usr/share/vim/vimfiles/template.spec
  augroup END
endif

if has("cscope") && filereadable("/usr/bin/cscope")
   set csprg=/usr/bin/cscope
   set csto=0
   set cst
   set nocsverb
   " add any database in current directory
   if filereadable("cscope.out")
      cs add $PWD/cscope.out
   " else add database pointed to by environment
   elseif $CSCOPE_DB != ""
      cs add $CSCOPE_DB
   endif
   set csverb
endif

" Switch syntax highlighting on, when the terminal has colors
" Also switch on highlighting the last used search pattern.
if &t_Co > 2 || has("gui_running")
  syntax on
  set hlsearch
endif
##开启相关插件
 filetype on
 filetype plugin on
 filetype indent on
 ##当文件在外部被修改时，自动更新该文件
 set autoread
 ##激活鼠标
 set mouse=a
 ##开启语法
 syntax enable
 ##设置字体
 set guifont=dejaVu\ Sans\ MONO\ 10
 ##设置配色
 colorscheme desert
 ##高亮显示当前行
 set cursorline
 hi cursorline guibg=#00ff00
 ##激活代码折叠
 set foldenable
 ##对文中的标志进行折叠
 set foldmethod=manual
 set foldcolumn=3
 ##设置为自动关闭折叠
 set foldclose=all
 nnoremap <space> @=(()foldclosed(line'.')) < 0) ? 'zc' : 'zo')<CR>
 ##使用空格来替换tab
 set expandtab
 ##设置所有的tab和缩进为4哥空格
 set tabstop=4
 ##设定<<和>>命令移动时的宽度为4
 set shiftwidth=4
 ##退格键一次性删掉4个空格
 set softtabstop=4
 set smarttab
 ##关闭自动缩进
 set autoindent

 set ai
 ##智能缩进
 set si
 ##自动换行
 set wrap
 ##设置软宽度
 set sw=4
 set wildmenu
 #显示标尺
 set ruler
 ##设置命令行的高度
 set cmdheight=1
 ##显示行数
 set nu
 set lz
 ##设置退格
 set backspace=eol,start,indent
 set whichwrap+=<,>,h,l
 ##设置魔术
 set magic
 ##关闭错误信息响铃
 set noerrorbells
 ##显示匹配的括号
 set showmatch
 set mat=2
 ##搜索时高高亮显示搜索到的内容
 set hlsearch
 ##搜索时不区分大小写
 set ignorecase
 ##编码设置
 set encoding=utf-8
 set fileencoding=utf=8
 set termencoding=utf-8
 ##开启新行时使用智能自动缩进
 set smartindent
 set cin
 set showmatch
 ##隐藏工具栏
 set guioptions-=T
 ##隐藏菜单栏
 set guioptions-=m
 ##显示状态栏（默认为1，表示无法显示状态栏）
 set laststatus=2
 ##解决粘贴不换行
 set pastetoggle=<F9>
 ##设置背景色
 set background=dark
 ##设置高亮相关
highlight Search ctermbg=black ctermfg=white guifg=white guibg=black
if &term=="xterm"
     set t_Co=8
     set t_Sb=^[[4%dm
     set t_Sf=^[[3%dm
endif

" Don't wake up system with blinking cursor:
" http://www.linuxpowertop.org/known.php
let &guicursor = &guicursor . ",a:blinkon0"    
```


|相关配置文件	|功能描述|
|----|---|
|.viminfo|	用户使用vim的操作历史|
|.vimrc	|当前用户vim的配置文件|
|/etc/vimrc	|系统全局vim配置文件|
|/usr/share/vim/vim74/colors/|	配色模板文件存放路径|

vim 常用指令：


vim有三个模式，普通模式（即用vim刚刚打开某个文件时的模式）；编辑模式（按i或o键进入）；命令行模式（也叫末行模式，通常是以":"开头，可以输入命令）

[普通模式]


* G：光标移动道最后一行
* gg：光标移动到第一行，等价于1gg或1G
* 0：数字0，表示将光标从所在位置移动到当前行的开头
* $：将光标所在位置移动当前行的结尾
* n <enter>：数字键+回车，表示将光标从当前位置向下移动n行
* ngg：将光标移动到n行
* H：将光标移动到当前窗口的第一行
* M：将光标移动到当前窗口的中间行
* L：将光标移动到当前窗口的最后一行
* h：和左方向键一样，光标左移一格
* j：下移一格
* k：上移一格
* l：右移一格
* /string：从光标位置开始向下搜索string
* ?string：从光标开始向上搜索string
* n：从光标位置开始向上重复前一个搜索的动作
* N：从光标位置开始向下重复前一个搜索的动作
* :g/A/s//B/g：把符合A的内容全部替换为B，斜线为分隔符，可以用@，#等替代
* :%/s/A/B/g：把符合A的内容全部替换为B，斜线为分隔符，可以用@，#等替代
* n1,n2s/A/B/gc：n1，n2为数字，表示在第n1行和第n2行间寻找A，且用B替换
* Yy：复制光标所在行
* nyy：复制从光标开始往下的n行
* P/p：p表示将已复制的数据粘贴到光标的下一行，P表示上一行
* dd：删除光标所在行
* ndd：删除从光标处开始的n行
* u：回复前一个执行过的操作
* .：重复前一个执行过的动作


[编辑模式]

* i：在当前光标所在处插入文字
* a：在当前光标所在位置的下一个字符出插入文字
* I：在当前所在行的行首第一个非空格字符处插入，和A相反
* A：在当前所在行的行尾最后一个字符处插入，和I相反
* O：在当前所在行的上一行处插入新的一行
* o：在当前所在行的下一行处插入新行
* ESC：退出编辑模式，回道普通模式


[命令行模式]

* :wq：退出并保存
* :wq!：退出并强制保存
* :q!：强制退出不保存
* :n1,n2, w filename：n1和n2为数字，表示将n1行到n2行的内容保存成filename这个文件
* :n1,n2 co n3：n1和n2为数字，表示将n1行到n2行的内容复制到n3位置下
* :n1,n2 m n3：表示将n1行到n2行内容挪到n3位置下
* :!command：暂时离开vim，到命令行里执行command命令。例如!ls /etc，此时没有退出vim
* :set nu：显示行号
* :set none：取消行号
* :vs filename：垂直分屏显示，同时显示当前文件和filename对应文件的内容
* :sp filename：水平分屏显示，同时显示当前文件和filename对应文件的内容
* :1+#+ESC：在可视块模式下（ctrl+v），一次性注释所选的多行
* :n1,n2s/#/ /gc：与上一句相反，取消注释
* Del：在可视块模式下，一次性删除所选内容
* r：在可视块模式下，一次性替换所选内容


拓展：

* ctrl+w+n：新建分屏
* ctrl+w+s：新建水平分屏
* ctrl+w+v：新建垂直分屏
* vim -On file1 file2... 
* vim -on file1 file2 ...
* 大O表示垂直分屏，小o表示水平分屏
* ctrl+w：切换分屏
* :only或者ctrl+w+o：关闭其他分屏
* ctrl+w+c：关闭当前分屏

