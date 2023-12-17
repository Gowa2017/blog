---
title: ntpd升级到最新4.2.8p9
toc: true
date: 2016-12-14 20:50:10
tags: [运维, Linux]
categories: [Linux/Unix]
---
时钟的同步对于很多对于要求实时性的应用来说是很重要的，特别是与应用与数据库的同步。所以一般都会以数据库所有服务器，其他所有客户端进行同步。但是其也是爆出一些安全漏洞的，所以进行更新是很必要的。
<!--more-->
# 查看版本

	ntpd --version
# 启动路径

	cat /etc/init.d/ntpd | grep prog=
> prog=ntpd	

	which  ntpd
> /usr/sbin/ntpd

# 备份原来的配置

	cd /etc && cp ntp.conf ntp.conf.bak
# 编译

	./configure --prefix=/usr --bindir=/usr/sbin  \
		--sysconfdir=/etc  --enable-clockctl \
		--docdir=/usr/share/doc/ntp-4.2.8p9 
	make
	make install && install -v -o ntp -g ntp -d /var/lib/ntp
	ntpq -c version
# 查看新版本

	ntpd --version
# 重新启动

	cd /etc && cp ntp.conf.bak ntp.conf	
	service ntpd restart
	ntpq -p
如果输出正常就证明升级成功了
