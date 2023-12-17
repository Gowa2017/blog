---
title: Android-Studio将一个项目以Module形式的引入及遇到的坑
categories:
  - Android
date: 2018-11-15 22:05:58
updated: 2018-11-15 22:05:58
tags: 
  - Android
---
公司买了蓝牙指纹设备，需要在APP上集成，设备方提供了一个测试的APP及相关的代码。想着手动集成有点类，要是能把整个项目直接以Module形式或者是以Jar包的形式来处理的话，那就完毕了。
<!--more-->
搜索了一下，还真有。但需要一步步来。

# 导入Module

在 Android Studio 上点击 **File -> New -> Import Module** 现在我们要导入的项目 **APP文件夹** 路径。我的项目我把新的Module 名称叫做 fgtitReader。 
# 导入Module build.gralde 修改
## 修改应用插件

将 	`apply plugin: 'com.android.application' `改为
`apply plugin: 'com.android.library'`


## 删除 applicationId

在我们导入的 fgtitReader 的 build.gradle 内，删除调 applicationId 设备。

# AndroidManifest.xml修改

将这个文件中的登录 Activity 改为普通的 Activity。

```xml
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
```

把这几行去掉。

# 遇到的问题

## 编译出错

编译的时候出错了：

```
Constant expression required 
Resource IDs cannot be used in a switch statement in Android library modules less... (⌘F1) 
Validates using resource IDs in a switch statement in Android library module. Resource IDs are non final in the library projects since SDK tools r14, means that the library code cannot treat these IDs as constants
```

哎哟，属于 library 库内的资源 ID 是不是 final 的，无法作为 switch 的 case 比较，而主模块中的就可以。

那么，以 `if ... else ...` 来替换吧。

把鼠标放在 case 语句上的时候，会出现一个感叹号，点击一下，就会出现一个替换语句。


# 编译

以命令

```gradle
./gradlew fgtitReader:assemble 
```

会打包出来一个 aar 文件。