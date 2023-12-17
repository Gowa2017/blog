---
title: adblockplus的匹配规则-PAC用到
categories:
  - Misc
date: 2018-12-03 09:11:02
updated: 2018-12-03 09:11:02
tags: 
  - PAC
---
自定义 PAC 规则会用到呢。

[原文地址](https://adblockplus.org/en/filter-cheatsheet)

<!--more-->
#例子

## 通过部分地址匹配

![](../res/adblock_plus_01.png)

1. Verbatim text 这部分必须出现在地址内
2. Wildcard charcter 通配符部分
3. Separator 分隔符。要么是一个分隔符，或者是地址的结束

上面的图片会匹配以下地址：

* http://example.com/banner/foo/img
* http://example.com/banner/foo/bar/img?param
* http://example.com/banner//img/foo

不会匹配以下地址：

* http://example.com/banner/img
* http://example.com/banner/foo/imgraph
* http://example.com/banner/foo/img.gif

## 通过域名匹配

![](../res/adblock_plus_02.png)

1. Domain name anchor 域名锚点。这后面的必须是域名
2. Verbatim text 。 地址中必须出现的域名
3. Separtor 分隔符。表示域名的结束 `^`

上图会匹配以下：

* http://ads.example.com/foo.gif
* http://server1.ads.example.com/foo.gif
* https://ads.example.com:8000/

不会匹配下面的地址：

* http://ads.example.com.ua/foo.gif
* http://example.com/redirect/http://ads.example.com/

## 匹配确切的地址
![](../res/adblock_plus_03.png)

1. 锚点开始符号 `|`
2. 必须出现的部分
3. 锚点结束 `|`

这会匹配：

* http://example.com/

而不会匹配：

* http://example.com/foo.gif
* http://example.info/redirect/http://example.com/

# 匹配规则中的选项

匹配规则中有很多选项来改变他们的行为。

![](../res/adblock_plus_04.png)

1. Address to be blocked 要匹配的地址
2. Option Selector 可选分隔符。指名后面跟随的是过滤选项。
3. Type Option 类型选项 定义了要匹配的类型。通常是  script/images 表示只有这两种类型才匹配。 `~` 表示非的意思。
4. Domain Option 域名选项。 限制过滤只在指定的域名上。可以使用  `~` 来取反。

上面的图片在满足下面的情况下匹配了 *http://ads.example.com/foo.gif*

1. 地址被加载为 script 或者  image
2. 从域名 example.com 加载（或者子域名），且不是从 foo.example.com 加载。


# 例外规则
例外规则的用处是及时匹配上了我们要屏蔽的内容，也可以允许其通过。

## 特定请求的例外
![](../res/adblock_plus_05.png)

1. Exception rule。以 `@@` 开始。
2. Address to be allowed 这和以上规则是一样的。不过是允许例外而已。
3. Type option 类型选项。

上面的规则的意思是，对于 ads.example.com/notbanner 非 script 不进行过滤。


## 整个站点例外
![](../res/adblock_plus_05.png)

只需要把 类型指定为 document 即可。

# 注释

以 `!` 开头的注释



