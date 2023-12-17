---
title: 关于Cocos2d-Lua使用NDK编译过程的解析
categories:
  - Cocos2d-X
date: 2019-02-07 21:56:30
updated: 2019-02-07 21:56:30
tags: 
  - Cocos2d-X
---
由于使用NDK编译打包安卓的时候出现了一些麻烦，所以来看一下整个编译过程是怎么样样的。一般来说是把引擎编译成动态库，然后在 Java 写的安卓代码内进行调用。Android NDK 是一组允许您将 C 或 C++（“原生代码”）嵌入到 Android 应用中的工具。 能够在 Android 应用中使用原生代码对于想执行以下一项或多项操作的开发者特别有用：

<!--more-->

# mk 文件

在项目的 app/jni 目录下，会有 Application.mk 和 Android.mk 两个文件。


- Android.mk：必须在 jni 文件夹内创建 Android.mk 配置文件。 ndk-build 脚本将查看此文件，其中定义了模块及其名称、要编译的源文件、版本标志以及要链接的库。  

- Application.mk：此文件枚举并描述您的应用需要的模块。 这些信息包括：    
用于针对特定平台进行编译的 ABI。  
工具链。  
要包含的标准库（静态和动态 STLport 或默认系统）。

# 一般工作流程

1. 编写 C/C++ 代码。
2. 编写 Android.mk,Application.mk 文件。
3. ndk-build 读取上面的两个文件，然后生成静态库或者共享库。
4. Java 代码内进行调用库内代码。

**NDK 的核心目的之一是让您将 C 和 C++ 源代码构建为可用于应用的共享库。
**

# Android.mk
Android.mk 文件位于项目 jni/ 目录的子目录中，用于向构建系统描述源文件和共享库。 它实际上是构建系统解析一次或多次的微小 GNU makefile 片段。 Android.mk 文件用于定义 Application.mk、构建系统和环境变量所未定义的项目范围设置。 它还可替换特定模块的项目范围设置。

Android.mk 的语法用于将源文件分组为模块。 模块是静态库、共享库或独立可执行文件。 可在每个 Android.mk 文件中定义一个或多个模块，也可在多个模块中使用同一个源文件。 构建系统只会将共享库放入应用软件包。 此外，静态库可生成共享库。

除了封装库之外，构建系统还可为您处理各种其他详细信息。例如，您无需在 Android.mk 文件中列出标头文件或生成的文件之间的显式依赖关系。 NDK 构建系统会自动为您计算这些关系。 因此，您应该能够享受到未来 NDK 版本中新工具链/平台支持的优点，而无需接触 Android.mk 文件。

## 基本知识

### LOCAL_PATH
*Android.mk* 文件必须首先定义 *LOCAL_PATH* 变量：

```makefile
LOCAL_PATH := $(call my-dir)
```

此变量表示源文件在开发树中的位置。在这里，构建系统提供的宏函数 my-dir 将返回当前目录（包含 Android.mk 文件本身的目录）的路径。

### CLEAR_VARS

下一行声明 CLEAR_VARS 变量，其值由构建系统提供。

```makefile
include $(CLEAR_VARS)
```

CLEAR_VARS 变量指向特殊 GNU Makefile，可为您清除许多 LOCAL\_XXX 变量，例如 LOCAL\_MODULE、LOCAL\_SRC\_FILES 和 LOCAL\_STATIC\_LIBRARIES。 请注意，它不会清除 LOCAL_PATH。此变量必须保留其值，因为系统在单一 GNU Make 执行环境（其中所有变量都是全局的）中解析所有构建控制文件。 在描述每个模块之前，必须声明（重新声明）此变量。

### LOCAL_MODULE

接下来，LOCAL_MODULE 变量将存储您要构建的模块的名称。请在应用中每个模块使用一个此变量。

```makefile
LOCAL_MODULE := hello-jni
```

每个模块名称必须唯一，且不含任何空格。构建系统在生成最终共享库文件时，会将正确的前缀和后缀自动添加到您分配给 LOCAL_MODULE 的名称。 例如，上述示例会导致生成一个名为 libhello-jni.so 的库。

### LOCAL_SRC_FILES

下一行枚举源文件，以空格分隔多个文件：

```makefile
LOCAL_SRC_FILES := hello-jni.c
```

LOCAL\_SRC\_FILES 变量必须包含要构建到模块中的 C 和/或 C++ 源文件列表。

最后一行帮助系统将所有内容连接到一起：

```makefile
include $(BUILD_SHARED_LIBRARY)
```

BUILD_SHARED_LIBRARY 变量指向 GNU Makefile 脚本，用于收集您自最近 include 后在 LOCAL_XXX 变量中定义的所有信息。 此脚本确定要构建的内容及其操作方法。

示例目录中有更复杂的示例，包括您可以查看的带注释的 Android.mk 文件。 此外，示例：native-activity 详细说明了该示例的 Android.mk 文件。 最后，变量和宏提供本节中变量的进一步信息。

### LOCAL_LDLIBS

链接外部库：

```makefile
LOCAL_LDLIBS    := -llog -landroid -lEGL -lGLESv1_CM
```

### LOCAL_STATIC_LIBRARIES

指明静态库名称。

```makefile
LOCAL_STATIC_LIBRARIES := android_native_app_glue
```


### import-module
```makefile
$(call import-module,android/native_app_glue)
```