---
title: vnc在CentOS的开启相关知识
categories:
  - [Linux/Unix]
date: 2018-07-18 10:02:22
updated: 2018-07-18 10:02:22
tags: 
  - Linux
  - VNC
  - CentOS
---
自己用才MAC，但是公司用的VPN是网御星云的，必须在windows下用，但是服务器配置还可以，单独用的话有点浪费。于是装了CentOS，KVM布置了一个虚拟机，然后准备开启VNC来支持。另外，所使用的idea也需要在图形化的界面下进行调试，也需要开启VNC。

# 原理

简单来说，就是我们在服务器上启动一个 vnc服务器，CentOS的叫做 **Xvnc**，此服务器会接受我们 我们的连接，然后监控窗口系统的信息发送给我们，再把我们的操作反馈到服务器的窗口信息。 所以，使用 vnc 的前提是，我们开启了窗口系统，怎么开启，这里不赘述，自行谷歌。


# 安装

先安装 gnome 桌面环境:

```
yum groupinstall "GNOME Desktop"
```


CentOS上的安装，非常简单：

```
yum install vncserver
```

就OK了，然后，直接在当前用户下执行命令：

```
vncserver
```
就OK了，但是这里有很多细节其实我们还不知道的。

# 配置

执行 `vncserver` 命令的时候，我们可以指定很多选项，如果不指定任何选项的话，会默认使用第一个可用的 显示ID，通常是 1。用这个显示ID启动  Xvnc，在Xvnc会话里启动窗口管理器。我们可以以 `vncserver :13`这样的形式指定显示ID。

1. `$HOME/.vnc/xstartup` 一个shell脚本，指定当VNC桌面开始时执行的 X 程序。如果这文件不存在，vncserver会建立一个默认的 xstartup 脚本，此脚本尝试 启动你选择的窗口管理器。
2. `/etc/tigervnc/vncserver-config-defaults` 这是一个可选的，系统层面的文件与`$HOME/.vnc/config`相等。如果此文件存在并定义了传递给Xvnc的选项，所有用户都会默认使用这些选项。`$HOME/.vnc/config`会重写这个文件内设置。文件的加载顺序是：`/etc/tigervnc/vncserver-config-defaults, $HOME/.vnc/config,  /etc/tigervnc/vncserver-config-mandatory`所有文件都不一定要求必须存在。
3. `/etc/tigervnc/vncserver-config-mandatory`系统层面与`$HOME/.vnc/config`想等的配置文件。如果这个文件定义了参数，那么就其会覆盖 `$HOME/.vnc/config`内定义的对应参数。
4. `$HOME/.vnc/config` 用户的配置文件
5. `$HOME/.vnc/passwd` 密码文件
6. `$HOME/.vnc/host:display#.log` 日志文件
7. `$HOME/.vnc/host:display#.pid` pid文件

# 以服务的形式启动CentOS 7

```
cp /lib/systemd/system/vncserver@.service /etc/systemd/system/vncserver@.service
```

把此文件中的 <USER> 一用户名替代 。

执行 `systemctl daemon-reload`

执行 `systemctl enable vncserver@<display>.service`

# 问题
以服务的形式启动失败了，最后是以命令的形式进行启动的。








