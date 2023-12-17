---
title: macOS用HomeBrew安装Sqlplus
categories:
  - macOS
date: 2018-10-26 13:17:43
updated: 2018-10-26 13:17:43
tags: 
  - Oracle
  - Sqlplus
  - brew
---
远程设备上用起来始终不太爽，调试起来麻烦，所以本地安装一个吧。[原文](https://vanwollingen.nl/install-oracle-instant-client-and-sqlplus-using-homebrew-a233ce224bf)
<!--more-->

# 下载

[http://www.oracle.com/technetwork/topics/intel-macsoft-096467.html. ](http://www.oracle.com/technetwork/topics/intel-macsoft-096467.html)从这网站下载两个文件这是必须的。因为甲骨文的授权问题。

1. instantclient-basic-macos.x64–11.2.0.4.0.zip
2. instantclient-sqlplus-macos.x64–11.2.0.4.0.zip

把这两个文件放到 *~/Library/Caches/Homebrew* 下

# 安装

```shell
brew tap InstantClientTap/instantclient
brew install instantclient-basic
brew install instantclient-sqlplus
```

执行过程出了错照着改就是了：

```shell
Error: The package file can not be downloaded automatically. Please sign in
and accept the licence agreement on the Instant Client downloads page:

  http://www.oracle.com/technetwork/topics/intel-macsoft-096467.html

Then manually download this file:

  http://download.oracle.com/otn/mac/instantclient/122010/instantclient-basic-macos.x64-12.2.0.1.0-2.zip

To this location (a specific filename in homebrew cache directory):

  /Users/wodediannao/Library/Caches/Homebrew/downloads/665aa2952dd4fcdbbe25f6a02ee3cc8cf5b39ab36c8001447b303fe567cc8354--instantclient-basic-macos.x64-12.2.0.1.0-2.zip

An example command to rename and move the file into the homebrew cache:

  $ cd /path/to/downloads && mv instantclient-basic-macos.x64-12.2.0.1.0-2.zip /Users/wodediannao/Library/Caches/Homebrew/downloads/665aa2952dd4fcdbbe25f6a02ee3cc8cf5b39ab36c8001447b303fe567cc8354--instantclient-basic-macos.x64-12.2.0.1.0-2.zip

Instead of renaming and moving you can create a symlink:

  $ cd /path/to/downloads && ln -sf $(PWD)/instantclient-basic-macos.x64-12.2.0.1.0-2.zip /Users/wodediannao/Library/Caches/Homebrew/downloads/665aa2952dd4fcdbbe25f6a02ee3cc8cf5b39ab36c8001447b303fe567cc8354--instantclient-basic-macos.x64-12.2.0.1.0-2.zip
```

 基本的原因就是必须要改成那种代码形式的文件名，放在download里面去。

# brew tap

这个命令经常在自定义下载的内容的时候用到。

选项参数：

```
brew tap [--full] [--force-auto-update] user/repo [URL]
```

实际上就是指定一个我们要下载的项的源的意思。

如果我们没有指定 *URL* 参数，那么会使用 HTTPS 从 github 来获取。因为 github 上托管了很多的 tap，这其实是命令 `brew   tap  user/repo https://github.com/user/homebrew-repo` 的一个简写。

如果指定了 *URL* 的话，那么我们就可以从任何地方来获取资源了，使用只要 git 能处理的传输协议。我们可以诸如 *SSH, GIT, HTTP, FTP(S), RSYNC* 这些协议来获取资源。

默认情况下，资源会作为  shadow copy( depth = 1) 来克隆，但如果我们指定 *-full* 参数的话，就会进行完整克隆。想要将一个 shadow copy 转换为 full copy ，指定 *-full* 参数 重新获取资源。