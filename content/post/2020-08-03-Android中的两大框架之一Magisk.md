---
title: Android中的两大框架之一Magisk
categories:
  - Android
date: 2020-08-03 08:21:06
updated: 2020-08-03 08:21:06
tags: 
  - Android
  - Magisk
---
安卓的世界里，除了 Xposed ，还有一个后起之秀 Magisk，他与 Xposed 的不他是，Xposed 是通过利用替换  app_process 这个程序，预先 C 层加载一些服务，然后对运行时做了一些修改来达成目的；而 Magisk 则是通过将一个文件系统， overlay ，类似分层的形式，来 merge  system 目录达成目的。不过，实现的过程中，其实手续会更多。
<!--more-->

# 一句话描述

> Magisk 是用来自定义安卓 4.2 版本以上系统的开源工具集。它覆盖了安卓自定义的基础部分：root, boot scripts, SELinux scripts, AVB 2.0/dm-verify/forceencrypt。

# 内部细节

## sbin tmpfs overlay

Magisk 一个突破性的设计就是 `sbin tmpfs overlay`（sbin 临时文件系统覆盖）。用来支持 system-as-root（将 /system 目录作为运行环境的根目录）设备是必须的，也是用来防止 Magisk 被检查的关键。所有的 Magisk 二进制程序，应用，镜像和其他尝试性的功能都被放在挂载在目录  /sbin 的 tmpfs 文件系统中。MagiskHide 可以简单的取消对于 /sbin 挂载和绑定挂载来隐藏所有的修改。

这里有一个关键，就是 `PATH` 变量包含了  `/sbin`，目录，所以这些程序放在这里，才能报找到。

关于在这个路径下的内容，可以查看 [官方文档](https://topjohnwu.github.io/Magisk/details.html#paths-in-sbin-tmpfs-overlay)

## /data 下的路径
某些二进制程序和文件需要存在在非易失性的存储设备的 `/data`中。为了阻止被检测，所有的东西都必须存在一个安全及不可检测的 `/data` 目录中。之所以选择了 `/data/adb` 目录是因为：

 - 在现代安卓上，它是一个已经存在的目录，因此不能用来作为证明 Magisk 存在的标志。
 - 这个目录的默认权限是 `700`，拥有者是 `root`，因此非 root 的进程是无法进入此目录，无法以任何途径来读写此目录。
 - 这目录被打上了 `u:object_r:adb_data_file:s0` 安全标签，因此很少几个进程可以与此这个安全上下文进行交互。
 - 这个目录位于 `Device encrypted storage`，因此当数据被正确的挂载到 FBE（File based Encryption），它就可以马上被访问了。

 # Magisk Booting 过程

 ## Pre-init

 程序 *magiskinit* 会先于 *init* 程序而执行，因此作为系统的第一个进程，它就可以干很多事情而不被检测了。它做了这些事情：

- 预先挂在 Magisk 需要的分区。在 system-as-root 的设备上，将 root 根目录切换至 system。
- 将 Magisk 服务注入到 `init.rc`
- 加载安全策略。从 */sepolicy*，或者是从厂商预先编译的安全策略，或者分散的安全策略。
- 为安全策略打补丁，并 dump 到 */sepolicy* 或者 */sbin/.se*，然后再为  *init* 或者 *liblinux.so* 打补丁来加载这些打了补丁的安全策略。
- 执行  *init*。

## post-fs-data

当 */data* 目录被正式的解密（如果需要）和挂在后，就会触发 post-fs-data 上的触发器。此时会启动 `magiskd` 守护进程，post-fs-data 脚本被执行，模块文件被挂载。

## late_start

引导过程的后期，类 `late_start` 会被触发， Magisk 的 *服务* 模式被启动。在这个模式下，服务脚本被执行，同时会在没有安装 Magisk 管理器的情况下安装 Magisk 管理器。

# 工具说明

```
magiskboot                 /* binary */
magiskinit                 /* binary */
magiskpolicy -> magiskinit
supolicy -> magiskinit
magisk                     /* binary */
magiskhide -> magisk
resetprop -> magisk
su -> magisk
```

> 在我们下载的 Magisk zip 文件中，只包含了 `magiskboot, magiskinit, magiskinit64`。magiskinit 被压缩和嵌入到了 `magiskinit64` 中。我们可以先把 magiskinit64 放到设备上了后，利用命令 `magiskinit64 -x magisk <path>` 来把它解压出来。

## magiskboot

一个用来解包/重新打包 boot 镜像，解析/打补丁/解压 cpio，打补丁 dbt，16进制的为二进制打补丁，和以多种算法来压缩/解压缩文件的工具。

*magiskboot* 的概念是让修改 boot 镜像更简单。

## magiskinit

这个工具是用来替换位于已打补丁的 boot 镜像中 ramdisk 内的 init 程序。

## magiskpolicy

有一个别名， supolicy，用来支持 SuperSu 的安全工具。

它是 *magiskinit* 的一个应用。主要是用来进行为安全策略打补丁。

所有，从 Magisk 守护进程 fork 的进程，包括 root shells，都在 *u:r:magisk:s0* 这个上下文运行。所有安装了 Magisk 的系统使用的安全策略都可以被看成是 *sepolicy* 和这些补丁的集合： `magiskpolicy --magisk 'allow magisk * * *`

## magisk

当 magisk 二进制程序以名字 *magisk* 进行调用的时候，它是命令工具使用的，拥有很多的帮助函数，也包含了很多 magisk 服务的入口。

## su

*magisk* 的一个应用，MagiskSU 的入口。

## resetprop

*magisk* 的一个应用。高级系统属性操纵工具。

## magiskhide

*magisk* 的一个应用，用来控制 MagiskHide 的命令行工具。用这个工具与Magisk 守护进程通信来改变 MagiskHide 的设置。

# 安装

这个其实就是需要将 boot.img 进行打补丁来将 magisk 需要的内容写到 ramdisk 里面去，这样的话才能在启动的时候执行 magiskinit。

## boot.img 

我们可以随便打开一个 boot.img 来看一下，工具的话我们使用 [XDA-发布的工具集](https://forum.xda-developers.com/showthread.php?t=2319018)

```sh
./split_boot boot.img 
```

可以看到里面的文件内容：

```
boot.emmc.win-kernel
boot.emmc.win-ramdisk.cpio.gz
ramdisk
```
包含了一个 kernel 文件，还有一个 cpio.gz 打包的文件系统。

magisin 对 ramdisk 打补丁，就是替换掉 ramdisk 里面的 init 文件为 magiskinit，同时还会动态的修改里面的 *init.rc* 文件。

## 模块目录

安装后的模块位于 `/data/adb/modules` 目录下。/data/adb 目录下还有很多的其他工具集。