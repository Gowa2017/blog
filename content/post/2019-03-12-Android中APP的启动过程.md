---
title: Android中APP的启动过程
categories:
  - Android
date: 2019-03-12 23:03:24
updated: 2019-03-12 23:03:24
tags: 
  - Android
---
安卓系统实质上是 Linux 内核的，其在 init 进程启动后，会启动 Zygote 进程。这个进程，会开启第一个 Java VM，预先加载很多与安卓系统框架相关的以及通用的一些资源。接着就会开个套接字来监听请求，根据请求来开启新的进程，VM来执行APP。一旦收到新的请求, Zygote会基于自身预先加载的VM来孵化出一个新的VM创建一个新的进程。





<!--more-->
启动Zygote之后, init进程会启动runtime进程. Zygote会孵化出一个超级管理进程---System Server. SystemServer会启动所有系统核心服务, 例如Activity Manager Service, 硬件相关的Service等. 到此, 系统准备好启动它的第一个App进程---Home进程了.

# Zygote 

1. init 进程及其他内核进程启动后，就会执行 `/system/bin/app_process`（源代码：[https://github.com/android/platform_frameworks_base/blob/master/cmds/app_process/app_main.cpp](https://github.com/android/platform_frameworks_base/blob/master/cmds/app_process/app_main.cpp)。事实上，这个程序调用的是`AndroidRuntime.start()`（源代码：[https://github.com/android/platform_frameworks_base/blob/master/core/jni/AndroidRuntime.cpp](https://github.com/android/platform_frameworks_base/blob/master/core/jni/AndroidRuntime.cpp)，参数：`com.android.internal.os.ZygoteInit, start-system-server`
2. `AndroidRuntime.start()` 启动 Java VM，调用 `ZygoteInit.main()`[https://github.com/android/platform_frameworks_base/blob/master/core/java/com/android/internal/os/ZygoteInit.java](https://github.com/android/platform_frameworks_base/blob/master/core/java/com/android/internal/os/ZygoteInit.java)，参数是 *start-system-server。*
3. `ZygoteInit.main()` 首先注册套接字（根据从套接字上收到的请求来 fork 进程）。接着就会预加载很多的类及许多的 xml, drawable 资源。接着调用 `startSystemServer()` fork 一个新进程来执行 *com.android.server.SystemServer* [https://github.com/android/platform_frameworks_base/blob/master/services/java/com/android/server/SystemServer.java](https://github.com/android/platform_frameworks_base/blob/master/services/java/com/android/server/SystemServer.java)
4. SystemServer fork 后，`runSelectLoopMode()` 被调用。这是一个`while(true)`的循环，从监听的套接字上获取请求。一旦有命名到达，就会调用 `ZygoteConnection.runOnce()`[https://github.com/android/platform_frameworks_base/blob/master/core/java/com/android/internal/os/ZygoteConnection.java](https://github.com/android/platform_frameworks_base/blob/master/core/java/com/android/internal/os/ZygoteConnection.java)。
5. `ZygoteConnection.runOnce() `调用 `Zygote.forkAndSpecialize()` [https://github.com/CyanogenMod/android_libcore/blob/gingerbread/dalvik/src/main/java/dalvik/system/Zygote.java](https://github.com/CyanogenMod/android_libcore/blob/gingerbread/dalvik/src/main/java/dalvik/system/Zygote.java)来　fork 进程。


在启动 SystemServer 的过程中，会启动一个叫做 ActivityManagerService 的核心服务。用来管理我们的 Activity。[https://android.googlesource.com/platform/frameworks/base/+/4f868ed/services/core/java/com/android/server/am/ActivityManagerService.java](https://android.googlesource.com/platform/frameworks/base/+/4f868ed/services/core/java/com/android/server/am/ActivityManagerService.java)。这个时候，我们的启动器，也就是桌面，也已经启动了。

事实是，当我们用 startActivity(intent) 这样的形式来启动 app 的时候，最终，请求都是发送给 ActivityManagerService 的。

根据 intent 解析出对应的 ActivityInfo, ApplicationInfo，进行构造启动。
# 参考

参考：

[http://multi-core-dump.blogspot.com/2010/04/android-application-launch.html](https://link.jianshu.com/?t=http://multi-core-dump.blogspot.com/2010/04/android-application-launch.html)  
[https://link.jianshu.com/?t=http://multi-core-dump.blogspot.com/2010/04/android-application-launch-part-2.html](https://link.jianshu.com/?t=http://multi-core-dump.blogspot.com/2010/04/android-application-launch-part-2.html)  
[http://liuwangshu.cn/framework/booting/3-syetemserver.html](http://liuwangshu.cn/framework/booting/3-syetemserver.html)