---
title: 关于在Module中添加aar包无效的问题
categories:
  - Android
date: 2019-05-10 22:04:50
updated: 2019-05-10 22:04:50
tags: 
  - Android
---
情况是因为要接入各种 SDK，大多都是以 aar 包的形式来提供的。所以想着将所有的 aar 包放在一个 module 内，其他 module 依赖这个 aar 包就行了。但显然，事实是残酷的，我想得太简单了呢。

<!--more-->

为了偷懒，在大华 SDK DEMO 中他写了一些简单的逻辑和界面，我也不想自己重新写了，就直接将 DEMO 以 module 的形式进行了集成。下面就开始这个过程。

# 关于 flatDir

当我们要指定本地某个目录内寻找依赖的时候，我们一般会利用 flatDir 来将目录添加到 repositories：

```groovy
repositories {
    flatDir {
        dirs 'lib'
    }
    flatDir {
        dirs 'lib1', 'lib2'
    }
}
```

关于资源类型的说明，可以参考：

[Repository Types](https://docs.gradle.org/current/userguide/repository_types.html)

# 回顾过程
## 创建 module aars

创建一个名为 aars 的模块，将所有的 aar 包都放在 libs 目录下。

aars/build.gradle：配置如下

```groovy
apply plugin: 'com.android.library'

android {
    compileSdkVersion Integer.parseInt(COMPILE_SDK_VERSION)


    defaultConfig {
        minSdkVersion Integer.parseInt(MIN_SDK_VERSION)
        targetSdkVersion Integer.parseInt(TARGET_SDK_VERSION)
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"

    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation fileTree(dir: 'libs', include: ['*.aar'])
    implementation "com.android.support:appcompat-v7:${SUPPORT_LIB_VERSION}"
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'com.android.support.test:runner:1.0.2'
    androidTestImplementation 'com.android.support.test.espresso:espresso-core:3.0.2'
}

```

## 依赖 aars 的 module

新建了一个 dhsdk 模块，依赖于 aars 内的很多内容。很天真的直接就添加了依赖：

dhsdk/build.gradle:

```groovy
apply plugin: 'com.android.library'

android {
    compileSdkVersion Integer.parseInt(COMPILE_SDK_VERSION)
    buildToolsVersion BUILDTOOLS_VERSION
    defaultConfig {
        minSdkVersion Integer.parseInt(MIN_SDK_VERSION)
        targetSdkVersion Integer.parseInt(TARGET_SDK_VERSION)
        versionCode 6
        versionName "1.6"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}

dependencies {
    implementation fileTree(include: ['*.jar'], dir: 'libs')
    implementation "com.android.support:appcompat-v7:${SUPPORT_LIB_VERSION}"
    implementation "com.android.support:design:${SUPPORT_LIB_VERSION}"
    implementation "com.android.support:recyclerview-v7:${SUPPORT_LIB_VERSION}"
    implementation 'com.github.CymChad:BaseRecyclerViewAdapterHelper:2.9.34'
    implementation project(':pulltofresh')
    implementation project(':aars')
}
```

结果是什么呢？在 dhsdk 这个模块中，根本就找不到在 aars 中添加的那些依赖的 aar。

## 依赖失败

这是因为：
**implementation** 所实现的依赖，对于其他模块，是不可见的。如果，我们想要将 aars 模块中的依赖的 aar 的内容都对依赖它的模块可见，需要用 **api** 来进行替换：

aars/build.gradle

```groovy
    api fileTree(dir: 'libs', include: ['*.aar'])
```

这样，我们的 dhsdk 就依赖成功了。

同理，如果我们要在 app 中想要依赖到 aars 中的内容，那么也必须让 dhsdk 以 api 的形式来依赖 aars。

# implementation 与 api

- implementation 声明完全内部的依赖，不会暴露到消费者。
- api 声明完全导出到消费者的依赖。

