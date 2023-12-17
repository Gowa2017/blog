---
title: SYSLOG审计日志配置
date: 2016-10-31 13:42:08
categories: [Linux/Unix]
toc: true
tags: [Linux, Syslog, 运维]
---
## 前言

syslog是UNIX系统中提供的一种日志记录方法(RFC3164)，syslog本身是一个服务器，程序中凡是使用syslog记录的信息都会发送到该服务器，服务器根据配置决定此信息是否记录，是记录到磁盘文件还是其他地方，这样使系统内所有应用程序都能以统一的方式记录日志，为系统日志的统一审计提供了方便。
AIX上同样提供这套服务。
<!--more-->
## 服务与配置
默认情况下，配置文件位于**/etc/syslog.conf**。但我们可以用-f选项来指定程序启动时读取的配置文件。
AIX使用

	startsrc -s syslogd
	stopsrc -s syslogd

来启动或者停止相应服务。
## 配置文件
配置文件用来告诉syslogd服务将哪些程序产生的，什么等级的信息，写入到什么地方。
每行配置文件类似以下格式：

	selection	action	rotation
	<msg_src_list> <destination> [rotate [size <size> k|m] [files <files>] [time <time> h|d|w|m|y] [compress] [archive <archive>]]

### 程序选择：

	kern	内核
	user	用户等级
	mail	邮件子系统
	daemon	守护程序
	auth	安全或授权
	syslog	syslogd守护程序
	lpr	行式打印机
	news	新闻子系统
	uucp	uucp子系统
	local0-7	本地使用
	*	所有程序


### 优先级
emerg	 指定紧急消息（LOG_EMERG）。这些消息并非分发给所有用户。可以将 LOG_EMERG 优先级消息记录到单独文件备查。
alert	 指定重要的消息（LOG_ALERT），如严重的硬件错误。这些消息分发给所有用户。
crit	 指定不列为错误的关键消息（LOG_CRIT），如不适当的登录尝试。LOG_CRIT 和较高优先级消息会发送到系统控制台。
err	 指定表示错误情况的消息（LOG_ERR），例如失败的磁盘写入。
warning	 指定反常但可恢复的情况的消息（LOG_WARNING）。
notice	 指定重要的参考消息（LOG_NOTICE）。没有指定优先级的消息会映射为此优先级的消息。
info	 指定参考消息（LOG_INFO）。这些消息可以废弃，但它们在分析系统是很有用。
debug	 指定调试消息（LOG_DEBUG）。这些消息可以废弃。
none	 排除选定的程序。只有在同一 selector 字段里跟在带有 *（星号）的条目后时，该优先级级别才有用。

## 例子
** 1. 要在调试级别或更高级别将所有的邮件工具消息记录到文件 /tmp/mailsyslog，请输入以下命令：  **

	mail.debug /tmp/mailsyslog
** 2. 要将除来自邮件工具之外的所有系统消息发送到主机 rigil，请输入以下命令：  **

	*.debug;mail.none @rigil
** 3. 要将所有工具中优先级为 emerg 的消息和邮件工具与守护程序工具中优先级为 crit 及更高级别的消息发送到用户 nick 和 jam，请输入以下命令：  **

	*.emerg;mail,daemon.crit nick, jam
** 4. 要将所有邮件工具消息发送到所有用户的终端屏幕，请输入以下命令：  **

	mail.debug *
** 5. 要将调试级别或更高级别的工具消息记录到 /tmp/syslog.out，并且在下列情况下旋转文件：文件超过 500 KB，一周之后将旋转文件数限制为 10，使用压缩并使用 /syslogfiles 作为归档目录，请输入以下命令：  **

	*.debug /tmp/syslog.out rotate size 500k time 1w files 10 compress archive /syslogfiles  <S-Del>
