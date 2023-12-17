---
title: CentOS system install and setup
date: 2016-10-31 12:05:57
tags: [CentOS, Linux]
categories: 
  - [Linux/Unix]
toc: true
---
大多数时候我们需要安装系统并进行一下配置，乃至加固等，这就记录一下，下次就不弄到处去找来给人了。
包括LVM建立、服务启动设置、SSH设置、网卡配置、源配置
<!--more-->
## 1安装时分区建立
1. 建立/boot分区 500M /swap 10000M
2. 剩余空间建立PV
3. PV加入VG
4. 建立LV 全部挂在到/
## 2安装模式选择
基本安装 开发工具全部装，SERVER 全部不选，中文语言支持装上，记得gcc一定要安装
## 3设置DVD源
	mkdir /media/dvd
	mount /dev/cdrom /media/dvd
	cd /etc/yum.repos.d
	mv CentOS-Base.repo CentOS-Base.repo.bak
** 修改 baseurl 增加**

	file:///media/dvd
设置enable=1

	yum clear all
## 4服务开启
	chkconfig --level 35 ntpd on
	chkconfig --level 35 sshd on
	chkconfig --level 35 network on
	chkconfig --level 35 iptables off
	chkconfig --level 35 networkmanager off
	service networkmanager stop
## 5禁止root登录
	sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
## 6增加用户并修改密码
	groupadd lcims
	useradd -d /lcims -s /bin/bash -g lcims lcims
	chown lcims.lcims /data
	passwd lcims
## 7修改显示模式
	echo "export PS1='[\u@\h $PWD]# '" >> /etc/profile
	source /etc/profile
## 8主机名
	vim /etc/sysconfig/network
## 9双网卡配置

1. 检查是否支持（默认情况下是支持的)

	cat /boot/config-2.6.32-431.el6.x86_64 | grep -i CONFIG_BONDING
config-XX是内核版本号 出现 CONFIG_BONDING=m说明支持，继续下一步，否则要进行编译内核支持。
2. 备份网卡设置

	cd /etc/sysconfig/network-scripts
	cp ifcfg-eth0 ifcfg-eth0.bak
	cp ifcfg-eth1 ifcfg-eth1.bak
3. 建立bond0网卡

	touch ifcfg-bond0
4.  网卡配置

	DEVICE="bond0"
	BOOTPROTO=static
	NM_CONTROLLED=no
	ONBOOT=yes
	TYPE=Ethernet
	IPADDR=10.11.189.66
	NETMASK=255.255.224.0
	GATEWAY=10.11.189.94
	IPV6INIT=no
	USERCTL=no
	BONDING_OPTS="mode=1 miimon=50"
5.  修改eth0/eth1网卡配置

	DEVICE="eth0"
	BOOTPROTO=none
	ONBOOT=yes
	TYPE=Ethernet
	USERCTL=no
	MASTER=bond0
	SLAVE=yes

6.  模块加载配置

	echo "alias bond0 bonding" >> /etc/modprobe/dist.conf
7.  网卡顺序设置

	echo "ifenslave bond0 eth0 eth1" >> /etc/rc.d/rc.local
	echo "touch /var/lock/subsys/local" >> /etc/rc.d/rc.local
8. 重启网络服务

	service network restart
9. 验证MAC地址是否一致

	ifconfig -a | grep HW
10. 运行状态检查

	cat /proc/net/bonding/bond0
