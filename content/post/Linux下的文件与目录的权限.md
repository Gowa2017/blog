---
title: Linux下的文件与目录的权限
toc: true
date: 2016-11-19 16:27:53
tags: [Apue, Linux]
categories: 
  - [Linux/Unix]
---
前些天遇到有朋友在群内讨论权限的问题，不是很清楚，为了几个比较细的问题，我一时也记得不太清楚。只记得需要对目录写入文件的话需要w权限，后面看了才确认还需要执行权限。
<!--more-->
# 从APUE看起
记得我在看APUE的时候有看到过关于目录项与文件像的说明，所以重新回去看了一下。
请看 __APUE 4.14节 文件系统__ 
#  i-node(i 节点)
我们可以把一个硬盘分为几个分区，每个分区都可以包含一个文件系统，那么这个文件系统的结构组成可能是以下这样的，i-node是一些固定长度的单元：
![linux file system partitions](/res/20161118-unix-file-system.png)
更加细致的看一下cylinder group1中的i-nodes与data blocks。
![linux file system partitions](/res/20161118-unix-file-system2.png)
i-node包含的信息很多，包括文件类型、文件权限位、文件大小、指向data blocks的指针等。
data blocks包含常规的数据块data blocks与存放目录项的 directory blocks。
内核通过i-node信息来确定数据存放的位置，通过i-node中的type字段来确定存放的是一般数据还是目录。
# 例子
听起来有不知所云，多看两次可能也觉得迷糊，我们通过一个例子来说明一下。
我们可以通过 ls -il 命令来查看文件的i节点号。通过 -ldi来显示目录的i节点
```
[www@iZ23fz9kp5sZ data]$ ll -di / /etc /etc/passwd
     2 drwxr-xr-x 23 root root  4096 11-15 12:24 /
2064385 drwxr-xr-x 96 root root 12288 11-18 12:24 /etc
2066133 -rw-r--r--  1 root root  1914 11-18 11:45 /etc/passwd
```
通过挂载点信息找到 / 的i-node号为2。
读取i-node 2指向的 directory blocks，此blocks中会存放类似  2064385 etc/ 的目录项
读取 2064385 i-node的信息，根据type field字段知道这指向一个 directoryblocks。
读取2064385指向的 directory blocks，找到类似 2066133 passwd 的目录项
读取 2066133 i-node的信息，前往对应blocks读取数据。
# 总结
__参考APUE 4.5节__
对于权限位 rwx大家都知道是 读写执行，但对于目录来说，可能不太清楚，咱们举例说明。
对一个目录例如 /etc/ 如果我们想在其目录下创建文件，其实是对 etc/ 存放目录项的增加，所以，我们需要 w写权限
i-node中的r权限信息决定了我们是否能够对此node指向的directory blocks进行读操作。
x权限决定了我们是否能够通过这个目录。

