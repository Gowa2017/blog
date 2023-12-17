---
title: 安装Genymotion免费版模拟器进行调试
categories:
  - Android
date: 2019-11-30 21:43:49
updated: 2019-11-30 21:43:49
tags: 
  - Android
  - Genymotion
  - Arm Translator
---

安卓原生的模拟器，就有一点不好，对于是使用了 arm abi 的 so 库的程序，其是无法进行安装的，因为模拟器本身是基于 x86 架构的，当然，我们可以安装 arm 架构的模拟器，但是那个速度感人，我在 mbp 2019 款上都跑得慢死了，据说 genymotion 速度可以还能支持 arm 的 apk 所以来安装一下。

<!--more-->

遗憾的是，当前， genymotion 官方网站已经找不到下载免费版的地址了，在网络上找了好久，在找到一个地址：

[https://www.genymotion.com/fun-zone](https://www.genymotion.com/fun-zone)

不过，如果我们用 brew 可以直接用 brew 进行安装：

```sh
brew cask install genymotion
```

genymotion 是依赖于 virtualbox 的，所以我们还需要安装它。具体怎么安装就不说了。

安装之后，我们还需要安装 arm translator 来进行对于 arm 原生库的 apk 支持。


# root 权限

[参考 genymotion 官方文档](https://docs.genymotion.com/paas/7.0/18_Using_root_access.html#from-an-application)

默认就已经提供了 root 权限了

# Arm translator

[genymotion 因为法律原因不提供 ARM translation tools.](https://docs.genymotion.com/desktop/3.0/03_Virtual_devices/032_Deploying_an_application.html#applications-with-arm-code)

>Genymotion Desktop virtual devices architecture is x86 (32bits). If your application relies on ARM native code, you must install an ARM translation tool to make it work. The ARM translation tool must match your virtual device Android version. Once installed, reboot your virtual device using adb (see Configuring Genymotion - ADB for details) with the following command:
>For legal reasons, Genymobile cannot provide you with any ARM translation tools.

所以我们需要自己安装。

[github 上有人提供了 Genymotion Translation](https://github.com/m9rco/Genymotion_ARM_Translation) 我们选择对应的版本。

然后进行安装。

```sh
  1. adb shell
  2. cd /sdcard/Download/
  3. sh /system/bin/flash-archive.sh /sdcard/Download/Genymotion-ARM-Translation.zip
  4. adb reboot
```

这样就 Ok 了



# Exposed 框架

这分两种情况。对于  SDK 22 以下的，直接安装APK即可：[官方安装教程在这里](https://repo.xposed.info/module/de.robv.android.xposed.installer)

而对于 SDK 22 以上的，就需要先刷入框架，然后再安装管理器 APK，[官方教程也在这里 SDK22以上](http://forum.xda-developers.com/showthread.php?t=3034811)



## 框架下载

[下载地址](https://dl-xda.xposed.info/framework/)

根据SDK等级，架构类型下载相应的内容

这里还有一个 uninstaller ，主要是为了用来卸载框架使用的

## 下载管理器

在 [这个帖子的附件中](https://forum.xda-developers.com/showthread.php?t=3034811)

## 安装

对于 Genymotion，直接将 APK/ZIP 文件拖到模拟器上就可以安装 。

对于真机的话，可能我们需要 TWRP 这样的工具来进行刷入。