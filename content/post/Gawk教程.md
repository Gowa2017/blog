---
title: Gawk教程
toc: true
date: 2016-11-22 17:37:41
tags: [Linux, AWK]
categories: 
- [Linux/Unix]
---
awk与sed，两大神器，处理字符文本效率爆棚。一直有用，但是没有完整的记录，临时百度感觉不好，做一个完整系统的阅读，并进行info手册的翻译，方便时时回顾。
<!--more-->
# 前言
awk在很多系统上都有不同的实现，Gawk是awk在GNU/Linux的实现，其基本命令格式为：

      gawk [ POSIX or GNU style options ] -f program-file [ -- ] file ...
      gawk [ POSIX or GNU style options ] [ -- ] program-text file ...
 更加直观一些的格式是:

 	awk [ -F fs ] [ -v var=value ] [ 'prog' | -f progfile] [ file ... ]
 或者更简单一般的形式  
 	
 	awk [ -F fs ] 'pattern { action }'


 	
# 工作原理
 awk以行为单为进行处理输入，然后用分隔符**FS(field separator)**把每行分隔成不同的**域(field)**。默认情况下使用空白符作为分隔符，但我们可以指定为任何我们想用的分隔符。
 `$0`代表了整个行，`$1, $2 ...`代表了被分隔开的每个域。 
接下来就会根据匹配 **pattern**的进行 **action** 动作。

 	awk -F: '{print $0,$1,$2,$3}
 上面这个例子没有 **pattern** 代表着无条件执行 **action** 的意思。
 然后观察一下输入是什么情况。  

# 命令行选项
- -F fs | --field-separator fs 指定域分隔符  
- -f source-file | --file source-file 指定程序文件  
- -v var=value | --assign var=value 在执行程序前设置变量。多个变量用`awk -v foo=1 -v bar=2`来指定  
- -b | character-as-bytes 将所有输入输出都认为是单字节的。
- -d | --dump-variables[=file]  打印变量。不指定_file_的话，就会放在文件 _awkvars_ 内。
- -D[file] | --debug[=file] 


# 内建命令

