---
title: android使用poi生成文书出错解决
categories:
  - Java
date: 2018-07-20 09:04:40
updated: 2018-07-20 09:04:40
tags: 
  - Android
  - Java
  - Poi
  - Docx
---
本来事情一切正常，使用的是poi来生成docx的文档。但是偶然间不知道在什么版本的时候，生成的时候会报错。提示是某个类没有实现List接口，但是非常明显的，之前都正常，并没有做什么特别的操作为什么会出现这样的问题呢？
<!--more-->
# 具体情况

```
java.lang.IncompatibleClassChangeError: Class 'org.apache.xmlbeans.impl.store.Cur' does not implement interface 'java.util.List' in call to 'int java.util.List.size()' (declaration of 'org.apache.xmlbeans.impl.store.Saver' appears in /data/app/cn.nanming.smartenterprise-lkfuXmBwdO_QyMt5K6sRdg==/base.apk!classes3.dex)
```
就是这么一句简单的描述，app就挂了，按照字面意思，说的是Cur类并没有实现 List.size() 接口，但为什么之前都是正常的呢？

更坑爹的是，debug版本的apk文件没有任何问题，release版本的 开启允许　debug的版本也没有问题，压根就无法找出问题。

# 第一次猜测

通过对比前一版本正常的apk包来进行了对比，发现对于正常的包，所有的 xmlbeans 的类是封装在 class2.dex 内的，而不正常的包就封装分别把类打包在了 class2.dex/class2.dex/class2.dex 内。才出现这样的问题。

我怀疑是不是分包的问题，但是百度良久依然没有发现什么有价值的解决办法。我们可以保存在 主 class.dex内的类，但是却无法分别指定某些类归属于哪个包的。

最终放弃了这个办法。

# 第二次猜测

在谷歌上找到的相关问题症状，显示的都是因为重复类导致的。但通过在idea来寻找，并没有发现重复的类。而通过 包依赖来分析，依然也没有发现重复的不同版本的依赖。

# 第三种猜测

昨天偶然的时候，看到在 libs 下有两个jar包，一个是 poi-shadow， 一个是	poi-scratchpad-3.17-beta1.jar，而查阅 poi.apache.org 发现，后一个包是针对操作 2003格式的文档的，而我操作的是 2007 文档，有ooxml就够了于是就把它删除了，之后重新编译，居然就OK了。

# 最终

但我却依然无法知道具体是什么情况，导致了这样的错误发生。