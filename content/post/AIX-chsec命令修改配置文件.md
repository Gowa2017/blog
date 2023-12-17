---
title: AIX-chsec命令修改配置文件
date: 2016-10-31 13:40:50
categories: [Linux/Unix]
toc: true
tags: [Aix]
---
AIX的所有配置设置通过一个命令来进行更改配置文件中的键－值对，以达到修改配置的目的。如：group/user/limits/passwd等等。
每次都用编辑器去改实在是太LOW了。
<!--more-->

## 前言
AIX的所有配置设置通过一个命令来进行更改配置文件中的键－值对，以达到修改配置的目的。如：group/user/limits/passwd等等
## 命令格式

	chsec [-f file] [-s Stanza] [-a attribute = value ...]
## 可以进行修改的配置文件
/etc/security/environ
/etc/security/group
/etc/security/audit/hosts
/etc/security/lastlog
/etc/security/limits
/etc/security/login.cfg
/usr/lib/security/mkuser.default
/etc/nscontrol.conf
/etc/security/passwd
/etc/security/portlog
/etc/security/pwdalg.cfg
/etc/security/roles
/etc/security/rtc/rtcd_policy.conf
/etc/security/smitacl.user
/etc/security/smitacl.group
/etc/security/user
/etc/security/user.roles
/etc/secvars.cfg
### 补充说明Stanza参数限制  

1. 修改/etc/security下的environ、last、limists、passwd、user，必须是有效的用户名或default。
2. 修改groups时必须是有效的组名或者default。
3. 修改portlog时必须是有效的端口名。
4. 修改login.cfg时必须是有效的端口名、方法名或usw属性。
5. 修改portlog、login.cfg中不存在的节属性时，自动创建该节。
6. passwd文件中的passwd节不能用chsec修改，只能用passwd命令修改
7. 修改/usr/lib/security/mkuser.default时必须是admin或user。

## 例子
1.修改默认密码策略  

	chsec -f /etc/security/user -s default -a minlen=8 -a histexpire=12 -a umask=077 -a loginretries=6 -a histsize=5
2.禁止root直接登录  

	chsec -f /etc/security/user -s root -a rlogin=false
3.允许所有用户登录时间  

	chsec -f /etc/security/user -s defalut -a logintimes=:0800-1700
