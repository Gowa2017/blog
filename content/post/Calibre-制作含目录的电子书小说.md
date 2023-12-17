---
title: Calibre 制作含目录的电子书小说
toc: true
date: 2017-11-06 22:28:27
tags: [Kindle]
categories: [Kindle]
---
还是**mobi、azw3**格式的小说在Kindle上看着舒服，但是总是发现没有目录的情况，所以搜索了一下网络，来制作一下对应的目录。
<!--more-->
# 前提
`Calibre`怎么安装，就不多说了。
然后你还需要一个支持正则表达式的文本编辑器。`Windows`下推荐`Emeditor`，然后如果说Linux或者Unix下的话，直接用sed就好了。
# 处理
先看一下转换书籍的内容目录项。对于目录的定义其是使用**Xpath**进行定义的。具体的意义我不是很明白，但是照着做就好了，抽空在学习一下。**Xpath**是针对html代码和文件的，所以我们要用html的方式来进行标签我们要处理的内容。
比如我下载了一本小说的txt文件，ypjs.txt，修改一下扩展名为**ypjs.html**。
里面分为两级架构，卷-章模式。我就用`sed`进行了批量的操作。
```sh
sed -i bak 's/\(^第[一二三四五六七八九十百零]*卷.*$\)/# \1/' ypjs.txt
sed -i bak 's/\(^第[一二三四五六七八九十百零]*章.*$\)/## \1/' ypjs.txt
sed -i bak '/^ *$/d' ypjs
```
处理后的文本如下：
![](/res/20171106-Calibre-make-toc2.png)
Calibre设置如下：
![](/res/20171106-Calibre-make-toc.png)
然后开转换吧
# 结果
![](/res/20171106-Calibre-make-toc3.png)
大功告成
