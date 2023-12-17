---
title: RealVNC的VNC-Viewer连接到VM虚拟机出错的问题
categories:
  - [Linux/Unix]
date: 2019-06-02 22:46:30
updated: 2019-06-02 22:46:30
tags: 
  - VNC
---

OS: CentOS 7 x64

开启了 vncserver，对于 Linux 系统本身而言，用 VNC Viewer 连接没有问题。

问题出在我用 KVM 添加了一个虚拟机，配置好 VNC 连接后无法连接。报出的错误是：

```
RFB protocol error: invalid message type ...
```

通过搜索，找到了结果：

[看这里](https://redmine.ixsystems.com/issues/23977)


解决办法：

在 Options 设置中，将 Picture quality 设置为 High，设置为 Automatic 是无法识别。
