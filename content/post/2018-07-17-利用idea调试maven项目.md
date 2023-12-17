---
title: 利用idea调试maven项目
categories:
  - Java
date: 2018-07-17 16:37:41
updated: 2018-07-17 16:37:41
tags: 
  - Java
  - Maven
---
事情是这样的，之前曾经idea上调试过项目，但是出现一个很头疼的问题是，后面再调用哪个API都会直接跳转到到了登录界面。而如果将环境部署以后，则正常，谷歌良久，终于找到了解决办法，但是，之前的疑问依然没有解决。为什么开始的时候同样的操作是可以的，而后就不行了呢。

# 环境

* OS: CentOS 7.5 x86_64
* IDE: ideaU
* 开发语言: Java
* 框架：jeeplus
* 编译：maven->war
* 中间件： tomcat 7.0.94

开始的时候，直接导入项目，编译，然后在idea的 `Run/Debug Configration` 处添加好 tomcatserver local项目，指定好我的tomcat 路径，然后直接小臭虫就开始debug了。

可是后面换了个机器，重新装一一下系统就坑了，不行了。谷歌才找到了解决的办法

# 解决办法

ideaU -> edit configration -> 添加 maven -> command line 输入 `tomcat7:run` 然后保存好后。

点小臭虫居然就可以了，这是为什么呢？

# 另外一种办法

在 idea 有了 tomcat server 插件集成的情况下。
按如下步骤进行：

1. 选择 Edit Configurations
2. 点 + 号，添加一个 Tomcat Server Local 配置
3. 在新添加的配置的 标签 Server 下指定自己的 Tomcat 程序位置
4. 在 Deployment 标签下，点 + 号添加我们的 smart.war exploded 。
5. 运行即可


