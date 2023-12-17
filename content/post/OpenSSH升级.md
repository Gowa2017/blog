---
title: OpenSSH升级
toc: true
date: 2016-10-31 14:06:47
tags: [SSH, 运维, Linux]
categories:  [Linux/Unix]
---
SSH，运维人员都不会陌生，可以说天天都与之打交道。但是时不时爆出的漏洞也让人感到很无语，对于大批量的服务器的要让我做安全加固的时候，内心其实是崩溃的。如果不是非常必要的话，还是用iptables/ipsec等方式来进行包过滤吧，这升级，实在是太折腾人了，本文并不保证在你的机器上能没有毛病的升级。
<!--more-->
[OpenSSH官方安装文档][1]
[1]:http://ftp.openbsd.org/pub/OpenBSD/OpenSSH/portable/INSTALL
# 环境
- OS: CentOS 6.8 x64
- zlib: 1.1.4 or 1.2.1.2 版本上
- libcrypto: OpenSSL >=0.9.8f < 1.1.0
> LibreSSL/OpenSSL should be compiled as a **position-independent** library
>(i.e. with -fPIC) otherwise OpenSSH will not be able to link with it.
>If you must use a non-position-independent libcrypto, then you may need
>to configure OpenSSH --without-pie.  Note that because of API changes,
>OpenSSL 1.1.x is not currently supported.

# 1、开启 telnet 服务
## Linux

	yum install -y telnet-server telnet
	/etc/xinet.d/telnet 中的yes 修改为no
	service xinetd restart
## AIX

	lssrc -s inetd
	startsrc -t telnet
	stopsrc -t telnet

# 2、下载相关文件

	wget http://ftp.openbsd.org/pub/OpenBSD/OpenSSH/portable/openssh-7.4p1.tar.gz
	wget https://www.openssl.org/source/openssl-1.0.1u.tar.gz
	wget https://sourceforge.net/projects/libpng/files/zlib/1.2.8/zlib-1.2.8.tar.gz --no-check-certificate

# 3、编译安装
## 安装zlib

	tar zxvf zlib-1.2.8.tar.gz
	cd zlib-1.2.8
	./configure
	make && make install

## 安装openssl
**解压**

	tar zxvf openssl-1.0.1u.tar.gz
	cd openssl-1.0.1u
**快速开始**

	./config -fPIC shared zlib
	make
	make install
这将会把OpenSSL安装在默认位置（因为历史原因）`/usr/local/ssl`。
我们可以验证一下:

	ls -1 /usr/local/ssl
>bin
>certs
>include
>lib
>man
>misc
>openssl.cnf
>private

**安装在自定义位置**
如果想安装在一个自己想要的地方，可以像下面一样指定：

	./config -fPIC shared zlib --prefix=/usr/local --openssldir=/usr/local/openssl
	make
	make depend # if prompted then you must do so
	make test
	make install
**安装前后的不同**
正常情况下（Linux）系统本身的openssl位于`/usr/{bin,lib[64],include/openssl}`，为了我们使用到的是新安装的，我们还需要做一下动作

	mv /usr/bin/openssl /usr/bin/openssl.bak
	mv /usr/include/openssl /usr/include/openssl.bak
	ln -s /usr/local/ssl/bin/openssl /usr/bin/openssl
	ln -s /usr/local/ssl/include/openssl /usr/include/openssl
	echo "/usr/local/ssl/lib" >> /etc/ld.so.conf
	ldconfig -v

**关于安装后的目录**
- bin openssl二进制文件和一些其他程序
- include/openssl 在协同`libcrypto或libssl`编译程序的时候需要的头文件
- lib OpenSSL库文件
**关于编译的选项**
- --prefix=DIR 安装在**DIR/{bin,lib,include/openssl}**，配置文件会在**DIR/ssl**或者 `--openssldir`指定的目录。
- --openssldir=DIR OpenSSL文件的目录。如果**没有指定--prefix**，那么二进制文件也会安装在这个地方。
> --prefix --openssldir 1.0.2及以下版本，可以不用指定，默认是/usr/local/ssl。1.1.0以上则必须指定。请避免使用--prefix=/usr
- no-threads 编译的时候不使用多进程。
- threads 编译的时候使用多进程
- no-zlib 编译的时候不进行zlib压缩、解压缩支持
- zlib 编译支持zlib压缩、解压缩
- zlib-dynamic 类似`zlib`，只在需要的时候载入zlib库，只在允许共享库的操作系统。默认选择
- no-shared 不创建共享库文件
- shared 对静态库的补充，还会创建共享库文件。
**安装过程中遇到的问题**
编译的时候出现 `Bad Value`类似的错误，需要

	make clean
一下。
然后重新`./config shared zlib`，再进行编译就OK。
## 安装OpenSSH

**解压**       

	tar zxvf openssh-7.4p1.tar.gz
	cd openssh-7.4p1

**安装**   

	./configure --sysconfdir=/etc/ssh --with-pam
	make
	make install
	sed -i 's/^GSSAPI/#&/' /etc/ssh/sshd_config

这样会将OpenSSH二进制文件安装到`/usr/local/bin`，配置文件在`/usr/local/etc`，服务在`/usr/local/sbin`。
**安装在不同位置**
如果想安装在不同的位置，可以指定`--prefix`。

	./configure --prefix=/usr/opt \ # Will install OpenSSH in /usr/opt/{bin,etc,lib,sbin}
	--sysconfdir=/etc/ssh \ # will place the configuration files in /etc/ssh.
	make
	make install

# 4、修改启动文件并重启
vi /etc/init.d/sshd ，修改
SSHD=/usr/sbin/sshd 为 SSHD=/usr/local/sbin/sshd

	sed -i 's#sbin/sshd#local/&#' /etc/init.d/sshd
	/etc/init.d/sshd restart

# 5、 验证安装

	telnet 127.0.0.1 22
>验证 根据回显看是否成功

# 6、替换客户端

	mv /usr/bin/ssh   /usr/bin/ssh_bak
	ln -s /usr/local/bin/ssh /usr/bin/ssh
