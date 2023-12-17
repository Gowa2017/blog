---
title: 在Linux为开启SFTP
categories:
  - [Linux/Unix]
date: 2019-08-13 17:31:31
updated: 2019-08-13 17:31:31
tags: 
  - 运维
  - Linux
---
新技能GET，虽然不是什么麻烦的事情，但是不仔细的用过还是会走不少弯路，再加上网络上的教程都是不明不白的，不会让你知其所以然。

<!--more-->

SFTP，也就是 SSH File Tranfer Protocol，是 FTP 文件传输协议的安全版本。其在一个 SSH（安全 Shell） 数据流上进行数据传输和数据访问。

所以其前提是要有一个 SSH 连接，然后在此连接上进行数据传输而已了。

其中权限的控制用到了 Linux 本身的权限机制，而如何区分是一个 纯粹的 SSH 连接还是一个 SFTP 连接，就由我们对 SSH 服务端进行定义了。

事实上 SSHD 会在我们的配置下，使用了 chroot 这个系统 API 来切换我们登录用户的根目录，将其放进一个受限的区域内，而不能访问其他地方的数据。

但其有一个点需要提前知晓的就是：

>ChrootDirectory
Specifies the pathname of a directory to chroot(2) to after authentication. All components of the pathname must be root-owned directories that are not writable by any other user or group. After the chroot, sshd(8) changes the working directory to the user's home directory.

就是说，对于 chroot 使用的路径，所有者必须是 root 并且，其他权限位不能有任何可写权限。

# 用户

增加sftp 组及用户。

```bash
groupadd sftpuser
useradd hgypt -d /upload -g sftpuser  -s /sbin/nologin # 指定家目录，组，及启动shell（禁止登录）
```

# 目录设置

我们要将所有的 sftp 用户，到限制在 /data/interface 目录下，每个用户对应一个目录

以下命令 root 执行：

```sh
mkdir -p /data/interface/hgypt
chmod 711 -R /data # 所有目录所有者都是root ，同时全部取消写权限，只有执行权限
chmod 755 /data/interface/hgypt
# 设置一个 sftp 用户拥有的目录
mkdir -p /data/interface/hgypt/upload
chown hgypt:hgypt /data/interface/hgypt/upload
chmod 600 /data/interface/hgypt/upload # 只允许sftp 用户进行读
```
# ssh 配置

针对特定的用户组，执行 chroot 设置：

```conf
Match Group sftpusers
ChrootDirectory /data/interface/%u
ForceCommand internal-sftp
```

