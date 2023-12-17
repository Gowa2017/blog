---
title: Android-NDK中使用Cmake
categories:
  - Android
date: 2020-01-29 23:15:10
updated: 2020-01-29 23:15:10
tags: 
  - Android
  - NDK
  - CMake
---
Android 的 [NDK][1] （原生开发套件，Native Development Kit）是一套工具集，使我们能在 Android 应用中使用  C/C++ 代码，并且提供了很多的[平台库][2]（如 **libc, libm, libdl, liblog, c++库**）。我们可使用这些平台库管理原生 Activity 和访问物理设备组件，例如传感器和轻触输入。当我想在 Andriod 上使用 [graphviz][3] 的时候，就必须要用到 NDK 来使用已存的 graphviz 代码了。 

<!--more-->
# NDK 和工具
想要在安卓应用中编译和调试原生代码我们需要三个工具：

- NDK
- CMake。一款外部编译工具，可与 Gradle 搭配使用来编译原生库。如果您只计划使用 ndk-build，则不需要此组件
- LLDB：Android Studio 用于调试原生代码的调试程序

如何进行配置和安装这三款工具，可以参考：[安装和配置 NDK 和 CMake][4]，不过这个地址好像在新版的 Android stuido 中并不适用。

# 编译项目
使用 NDK 编译项目有三种方法：

 - 基于 make 的 [ndk-build][5]
 - [CMake][7]
 - [独立工具链][6]，用于与其他编译工具集成，或者与基于 `configure` 的项目搭配使用。

这里我只关心一下 CMake 的方式，因为其他两种的话，我并不打算使用，具体的用法参考对应文档就行了。

# CPU和架构

因为 C 代码 编译成二进制代码，是针对特定的 CPU 指令集的，所以说，在编译的时候，我们就要用对相应的工具链，才能编译出正常能跑的二进制代码。

> 不同的 Android 设备使用不同的 CPU，而不同的 CPU 支持不同的指令集。CPU 与指令集的每种组合都有专属的应用二进制接口 (ABI)。ABI 包含以下信息：
>
> - 可使用的 CPU 指令集（和扩展指令集）。
> - 运行时内存存储和加载的字节顺序。Android 始终是 little-endian。
> - 在应用和系统之间传递数据的规范（包括对齐限制），以及系统调用函数时如何使用堆栈和寄存器。
> - 可执行二进制文件（例如程序和共享库）的格式，以及它们支持的内容类型。Android 始终使用 ELF。如需了解详情，请参阅 [ELF System V 应用二进制接口](http://www.sco.com/developers/gabi/2001-04-24/contents.html)。
> - 如何重整 C++ 名称。如需了解详情，请参阅 [Generic/Itanium C++ ABI](http://itanium-cxx-abi.github.io/cxx-abi/)。
>
> 本页列举了 NDK 支持的 ABI，并且介绍了每个 ABI 的运行原理。
>
> ABI 还可以指平台支持的原生 API。如需影响 32 位系统的此类 ABI 问题列表，请参阅 [32 位 ABI 错误](https://android.googlesource.com/platform/bionic/+/master/docs/32-bit-abi.md)。

## 支持的 ABI

| ABI                                                          | 支持的指令集                                               | 备注                     |
| :----------------------------------------------------------- | :--------------------------------------------------------- | :----------------------- |
| [`armeabi-v7a`](https://developer.android.com/ndk/guides/abis#v7a) | armeabi<BR>Thumb-2<BR>VFPv3-D16                            | 与 ARMv5/v6 设备不兼容。 |
| [`arm64-v8a`](https://developer.android.com/ndk/guides/abis#arm64-v8a) | AArch64                                                    |                          |
| [`x86`](https://developer.android.com/ndk/guides/abis#x86)   | x86 (IA-32)<BR>MMX<BR>SSE/2/3<BR>SSSE3                     | 不支持 MOVBE 或 SSE4。   |
| [`x86_64`](https://developer.android.com/ndk/guides/abis#86-64) | x86-64<BR>MMX<BR>SSE/2/3<BR>SSSE3<BR>SSE4.1、4.2<BR>POPCNT |                          |

>**注意**：NDK 以前支持 ARMv5 (armeabi) 以及 32 位和 64 位 MIPS，但 NDK r17 已不再支持。

# CMake

Android NDK 支持使用 CMake 编译应用的 C 和 C++ 代码。本页讨论如何通过 Android Gradle 插件的 ExternalNativeBuild 或通过直接调用 CMake 将 CMake 用于 NDK。

> 注意：如果您使用的是 Android Studio，请转至[向您的项目添加 C 和 C++ 代码][8]，了解以下基础知识：向项目中添加原生源代码，创建 CMake 编译脚本，将您的 CMake 项目添加为 Gradle 依赖项，以及使用比 SDK 中更新的版本的 CMake。

CMake 实际上不是一个构建工具，其使用简单的平台和与编译无关的配置文件用来控制软件的编译过程，生成原生的 makefile 文件和工作区。比如我们可以用 CMake 来生成 make 的编译文件，或者是 ninja 的构建规则文件。
## CMake 工具链文件

NDK 通过[工具链文件][9]支持 CMake。工具链文件是用于自定义交叉编译工具链行为的 CMake 文件。用于 NDK 的工具链文件位于 NDK 中的 `$ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake`。对于我使用 `brew` 安装的 NDK，则位于 `/usr/local/share/android-ndk/build/cmake/android.toolchain.cmake`。

在调用 cmake 时，命令行会提供诸如 ABI、minSdkVersion 等编译参数。有关所支持参数的列表，参阅工具链参数部分。

当我们使用 gradle 进行编译的时候，会自动使用此工具链文件，而若我们是手动通过命令行进行编译的，则必须手动指定此工具链文件，如：

```sh
$ cmake \
        -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake \
        -DANDROID_ABI=$ABI \
        -DANDROID_NATIVE_API_LEVEL=$MINSDKVERSION \
        $OTHER_ARGS
```

## 工具链文件什么用？

CMake 使用一系列的工具来进行 **编译，链接**库和创建归档文件，以及其他一些驱动编译的任务。常规情况下， CMake 会使用宿主设备上的工具链。但是在交叉编译的情况下，我们就必须指定一个工具链文件来知会使用的编译器和其他工具的路径。

对于我们想要编译的安卓项目来说 ，当我们指定了项目使用的 API 等级（MINSDKVERSION），目标设备的架构（ABI），我们就能通过工具链文件知道调用相应的工具来进行编译了。

## android.toolchain.cmake
这个文件有点长，有 700 多行。其位于 AOSP 项目的 [这个链接](https://android.googlesource.com/platform/ndk/+/master/build/cmake/android.toolchain.cmake)，

### 查找 NDK 路径

```make
get_filename_component(ANDROID_NDK_EXPECTED_PATH
    "${CMAKE_CURRENT_LIST_DIR}/../.." ABSOLUTE)
if(NOT ANDROID_NDK)
  set(ANDROID_NDK "${ANDROID_NDK_EXPECTED_PATH}")
```
其中 `CMAKE_CURRENT_LIST_DIR` 就代表了我们当前处理的文件目录，当 cmake 在处理我们传递过去的工具链的时候，工具链文件的父目录之父目录，就被认为是期望的 ANDROID_NDK 目录，也就是我们设置的环境变量 `$ANDROID_NDK_HOME`。

### ANDROID_TOOLCHAIN

```make
# Compatibility for configurable variables.
# Compatible with configurable variables from the other toolchain file:
#         https://github.com/taka-no-me/android-cmake
# TODO: We should consider dropping compatibility to simplify things once most
# of our users have migrated to our standard set of configurable variables.
if(ANDROID_TOOLCHAIN_NAME AND NOT ANDROID_TOOLCHAIN)
  if(ANDROID_TOOLCHAIN_NAME MATCHES "-clang([0-9].[0-9])?$")
    set(ANDROID_TOOLCHAIN clang)
  elseif(ANDROID_TOOLCHAIN_NAME MATCHES "-[0-9].[0-9]$")
    set(ANDROID_TOOLCHAIN gcc)
  endif()
endif()
```
这个设置，其实是为了和其他工具链文件中的配置文件相兼容，如果其他工具链文件中已经配置了安卓相关应该使用的工具链，那么就使用其他工具链文件中配置的了。

### ANDROID_API

```make
if(ANDROID_ABI STREQUAL "armeabi-v7a with NEON")
  set(ANDROID_ABI armeabi-v7a)
  set(ANDROID_ARM_NEON TRUE)
elseif(ANDROID_TOOLCHAIN_NAME AND NOT ANDROID_ABI)
  if(ANDROID_TOOLCHAIN_NAME MATCHES "^arm-linux-androideabi-")
    set(ANDROID_ABI armeabi-v7a)
  elseif(ANDROID_TOOLCHAIN_NAME MATCHES "^aarch64-linux-android-")
    set(ANDROID_ABI arm64-v8a)
  elseif(ANDROID_TOOLCHAIN_NAME MATCHES "^x86-")
    set(ANDROID_ABI x86)
  elseif(ANDROID_TOOLCHAIN_NAME MATCHES "^x86_64-")
    set(ANDROID_ABI x86_64)
  elseif(ANDROID_TOOLCHAIN_NAME MATCHES "^mipsel-linux-android-")
    set(ANDROID_ABI mips)
  elseif(ANDROID_TOOLCHAIN_NAME MATCHES "^mips64el-linux-android-")
    set(ANDROID_ABI mips64)
  endif()
endif()
```

根据是否设置  **`ANDROID_TOOLCHAIN_NAME`** 变量来设置 **`ANDROID_ABI`**。通过这一节和上一节，我们可以看到，**`ANDROID_TOOLCHAIN_NAME`** 可以设置为类似：`arm-linux-androideabi-4.9` （使用 ndk-build）或 `arm-linux-androideabi-clang9` 这样。

### ANDROID_PLATFORM
```make
if(ANDROID_NATIVE_API_LEVEL AND NOT ANDROID_PLATFORM)
  if(ANDROID_NATIVE_API_LEVEL MATCHES "^android-[0-9]+$")
    set(ANDROID_PLATFORM ${ANDROID_NATIVE_API_LEVEL})
  elseif(ANDROID_NATIVE_API_LEVEL MATCHES "^[0-9]+$")
    set(ANDROID_PLATFORM android-${ANDROID_NATIVE_API_LEVEL})
  endif()
endif()
```
设置要使用的安卓平台库版本。


### 默认工具链与 ABI
默认情况下，如果我们不设置：`ANDROID_TOOLCHAIN` 和 `ANDROID_ABI`，那么会默认使用  clang 和 armeabi-v7a

```make
if(NOT ANDROID_TOOLCHAIN)
  set(ANDROID_TOOLCHAIN clang)
elseif(ANDROID_TOOLCHAIN STREQUAL gcc)
  message(FATAL_ERROR "GCC is no longer supported. See "
  "https://android.googlesource.com/platform/ndk/+/master/docs/ClangMigration.md.")
endif()
if(NOT ANDROID_ABI)
  set(ANDROID_ABI armeabi-v7a)
endif()
```

### ANDROID_PLATFORM
默认情况下，最小的平台支持库是 16，在与工具链文件同一目录下的 `platforms.cmake` 中进行定义：

```make
# platforms
set(NDK_PLATFORM_ALIAS_P "android-28")
set(NDK_PLATFORM_ALIAS_O-MR1 "android-27")
set(NDK_PLATFORM_ALIAS_L-MR1 "android-22")
set(NDK_MAX_PLATFORM_LEVEL "28")
set(NDK_MIN_PLATFORM_LEVEL "16")
set(NDK_PLATFORM_ALIAS_J-MR2 "android-18")
set(NDK_PLATFORM_ALIAS_J-MR1 "android-17")
set(NDK_PLATFORM_ALIAS_N-MR1 "android-24")
set(NDK_PLATFORM_ALIAS_L "android-21")
set(NDK_PLATFORM_ALIAS_M "android-23")
set(NDK_PLATFORM_ALIAS_25 "android-24")
set(NDK_PLATFORM_ALIAS_O "android-26")
set(NDK_PLATFORM_ALIAS_N "android-24")
set(NDK_PLATFORM_ALIAS_20 "android-19")
set(NDK_PLATFORM_ALIAS_K "android-19")
set(NDK_PLATFORM_ALIAS_J "android-16")

```
```make
if(NOT ANDROID_PLATFORM)
  message(STATUS "\
ANDROID_PLATFORM not set. Defaulting to minimum supported version
${NDK_MIN_PLATFORM_LEVEL}.")
```

### ANDROID_STL

 C++ 的 STL 默认使用静态库，且不支持四种库。

```make
if(NOT ANDROID_STL)
  set(ANDROID_STL c++_static)
endif()

if("${ANDROID_STL}" STREQUAL "gnustl_shared" OR
    "${ANDROID_STL}" STREQUAL "gnustl_static" OR
    "${ANDROID_STL}" STREQUAL "stlport_shared" OR
    "${ANDROID_STL}" STREQUAL "stlport_static")
  message(FATAL_ERROR "\
${ANDROID_STL} is no longer supported. Please switch to either c++_shared or \
c++_static. See https://developer.android.com/ndk/guides/cpp-support.html \
for more information.")
endif()

```

### try_compile()
导出变量给 cmake 的 `try_compile()` 使用。
cmake 在调用  `try_compile()` 进行测试文件是否可以编译的时候，会用到这些变量。

```make
set(CMAKE_TRY_COMPILE_PLATFORM_VARIABLES
  ANDROID_TOOLCHAIN
  ANDROID_ABI
  ANDROID_PLATFORM
  ANDROID_STL
  ANDROID_PIE
  ANDROID_CPP_FEATURES
  ANDROID_ALLOW_UNDEFINED_SYMBOLS
  ANDROID_ARM_MODE
  ANDROID_ARM_NEON
  ANDROID_DISABLE_FORMAT_STRING_CHECKS
  ANDROID_CCACHE)
```
### ANDROID_LLVM_TRIPLE

根据 ABI，来设置  **`ANDROID_LLVM_TRIPLE`** 这个变量代表了：CPU，操作系统，平台库版本 三元组。

同时还会设置 **`ANDROID_TOOLCHAIN_NAME`** 工具链的默认值。

```make
set(CMAKE_ANDROID_ARCH_ABI ${ANDROID_ABI})
if(ANDROID_ABI STREQUAL armeabi-v7a)
  set(ANDROID_SYSROOT_ABI arm)
  set(ANDROID_TOOLCHAIN_NAME arm-linux-androideabi)
  set(CMAKE_SYSTEM_PROCESSOR armv7-a)
  set(ANDROID_LLVM_TRIPLE armv7-none-linux-androideabi)
elseif(ANDROID_ABI STREQUAL arm64-v8a)
  set(ANDROID_SYSROOT_ABI arm64)
  set(CMAKE_SYSTEM_PROCESSOR aarch64)
  set(ANDROID_TOOLCHAIN_NAME aarch64-linux-android)
  set(ANDROID_LLVM_TRIPLE aarch64-none-linux-android)
elseif(ANDROID_ABI STREQUAL x86)
  set(ANDROID_SYSROOT_ABI x86)
  set(CMAKE_SYSTEM_PROCESSOR i686)
  set(ANDROID_TOOLCHAIN_NAME i686-linux-android)
  set(ANDROID_LLVM_TRIPLE i686-none-linux-android)
elseif(ANDROID_ABI STREQUAL x86_64)
  set(ANDROID_SYSROOT_ABI x86_64)
  set(CMAKE_SYSTEM_PROCESSOR x86_64)
  set(ANDROID_TOOLCHAIN_NAME x86_64-linux-android)
  set(ANDROID_LLVM_TRIPLE x86_64-none-linux-android)
else()
  message(FATAL_ERROR "Invalid Android ABI: ${ANDROID_ABI}.")
endif()

set(ANDROID_LLVM_TRIPLE "${ANDROID_LLVM_TRIPLE}${ANDROID_PLATFORM_LEVEL}")
```

### ANDROID_HOST_TAG 

此变量代表了当前进行编译的宿主机系统，比如我使用的是 macOS，那么就是 darwin-x86_64。

```make
if(CMAKE_HOST_SYSTEM_NAME STREQUAL Linux)
  set(ANDROID_HOST_TAG linux-x86_64)
elseif(CMAKE_HOST_SYSTEM_NAME STREQUAL Darwin)
  set(ANDROID_HOST_TAG darwin-x86_64)
elseif(CMAKE_HOST_SYSTEM_NAME STREQUAL Windows)
  set(ANDROID_HOST_TAG windows-x86_64)
endif()

if(CMAKE_HOST_SYSTEM_NAME STREQUAL Windows)
  set(ANDROID_TOOLCHAIN_SUFFIX .exe)
endif()
```

### 工具链路径

```make
set(ANDROID_TOOLCHAIN_ROOT
  "${ANDROID_NDK}/toolchains/llvm/prebuilt/${ANDROID_HOST_TAG}")
set(ANDROID_TOOLCHAIN_PREFIX
  "${ANDROID_TOOLCHAIN_ROOT}/bin/${ANDROID_TOOLCHAIN_NAME}-")
set(ANDROID_SYSROOT "${ANDROID_TOOLCHAIN_ROOT}/sysroot")
list(APPEND CMAKE_SYSTEM_LIBRARY_PATH
  "${ANDROID_SYSROOT}/usr/lib/${ANDROID_TOOLCHAIN_NAME}/${ANDROID_PLATFORM_LEVEL}")

set(ANDROID_HOST_PREBUILTS "${ANDROID_NDK}/prebuilt/${ANDROID_HOST_TAG}")

```

在我们的设置上，这几个变量分别是：

- `ANDROID_TOOLCHAIN_ROOT`：`/usr/local/share/android-ndk/toolchains/llvm/prebuilt/darwin-x86_64`
- `ANDROID_TOOLCHAIN_PREFIX`：`/usr/local/share/android-ndk/toolchains/llvm/prebuilt/darwin-x86_64/bin/i686-linux-android-`
- `ANDROID_SYSROOT`：`/usr/local/share/android-ndk/toolchains/llvm/prebuilt/darwin-x86_64/sysroot` 安卓平台库文件
- `ANDROID_HOST_PREBUILTS`：`/usr/local/share/android-ndk/prebuilt/darwin-x86_64`  这里面是一些宿主机会用到的库文件

### 设置编译器

```make
set(ANDROID_C_COMPILER
  "${ANDROID_TOOLCHAIN_ROOT}/bin/clang${ANDROID_TOOLCHAIN_SUFFIX}")
set(ANDROID_CXX_COMPILER
  "${ANDROID_TOOLCHAIN_ROOT}/bin/clang++${ANDROID_TOOLCHAIN_SUFFIX}")
set(ANDROID_ASM_COMPILER
  "${ANDROID_TOOLCHAIN_ROOT}/bin/clang${ANDROID_TOOLCHAIN_SUFFIX}")
# Clang can fail to compile if CMake doesn't correctly supply the target and
# external toolchain, but to do so, CMake needs to already know that the
# compiler is clang. Tell CMake that the compiler is really clang, but don't
# use CMakeForceCompiler, since we still want compile checks. We only want
# to skip the compiler ID detection step.
set(CMAKE_C_COMPILER_ID_RUN TRUE)
set(CMAKE_CXX_COMPILER_ID_RUN TRUE)
set(CMAKE_C_COMPILER_ID Clang)
set(CMAKE_CXX_COMPILER_ID Clang)
set(CMAKE_C_COMPILER_VERSION 8.0)
set(CMAKE_CXX_COMPILER_VERSION 8.0)
set(CMAKE_C_STANDARD_COMPUTED_DEFAULT 11)
set(CMAKE_CXX_STANDARD_COMPUTED_DEFAULT 14)
set(CMAKE_C_COMPILER_TARGET   ${ANDROID_LLVM_TRIPLE})
set(CMAKE_CXX_COMPILER_TARGET ${ANDROID_LLVM_TRIPLE})
set(CMAKE_ASM_COMPILER_TARGET ${ANDROID_LLVM_TRIPLE})
set(CMAKE_C_COMPILER_EXTERNAL_TOOLCHAIN   "${ANDROID_TOOLCHAIN_ROOT}")
set(CMAKE_CXX_COMPILER_EXTERNAL_TOOLCHAIN "${ANDROID_TOOLCHAIN_ROOT}")
set(CMAKE_ASM_COMPILER_EXTERNAL_TOOLCHAIN "${ANDROID_TOOLCHAIN_ROOT}")
set(ANDROID_AR "${ANDROID_TOOLCHAIN_PREFIX}ar${ANDROID_TOOLCHAIN_SUFFIX}")
set(ANDROID_RANLIB
  "${ANDROID_TOOLCHAIN_PREFIX}ranlib${ANDROID_TOOLCHAIN_SUFFIX}")

```
这个没啥说的，就是设置编译器为 NDK 自带的 clang/clang++ 了。
还设置了 `CMAKE_C_COMPILER_TARGET , CMAKE_C_COMPILER_EXTERNAL_TOOLCHAIN,ANDROID_AR, ANDROID_RANLIB `。

## 正确指定架构

我们有两种方式来确保我们使用正确的架构进行构建。

1. 调用  Clang 是指定 `-target` 参数。
2. 调用具有前缀的 Clang

例如，若要为 64 位 ARM Android 编译值为 21 的 `minSdkVersion`，可使用以下两种方法，您可以选择使用其中最方便的方法：

```sh
$ $NDK/toolchains/llvm/prebuilt/$HOST_TAG/clang++ \
    -target aarch64-linux-android21 foo.cpp
```

```sh
$ $NDK/toolchains/llvm/prebuilt/$HOST_TAG/aarch64-linux-android21-clang++ \
    foo.cpp
```

这里的 **前缀** 是一个三元组：

| ABI         | 三元组                     |
| :---------- | :------------------------- |
| armeabi-v7a | `armv7a-linux-androideabi` |
| arm64-v8a   | `aarch64-linux-android`    |
| x86         | `i686-linux-android`       |
| x86-64      | `x86_64-linux-android`     |

> **注意**：对于 32 位 ARM，编译器会使用前缀 `armv7a-linux-androideabi`，但 binutils 工具会使用前缀 `arm-linux-androideabi`。对于其他架构，所有工具的前缀都相同。
>
> **因为这个原因，编译有的项目会出现问题**

许多项目的构建脚本都预计使用 GCC 风格的交叉编译器，其中每个编译器仅针对一种操作系统/架构组合，因此可能无法正常处理 `-target`。在这些情况下，最好使用三元组前缀的 Clang 二进制文件。

# Autoconf 项目

对于这种项目，因为 Autoconf 支持指定相关的编译工具链，所以我们可以手动进行指定：

- target 目标架构三元组
- binutils 相关的变量

例如：

```sh
# Check out the source.
git clone https://github.com/glennrp/libpng
cd libpng
# 指定宿主机编译工具链所在路径，根据宿主机来选择，如是 macos 或者是 x86的
export TOOLCHAIN=$NDK/toolchains/llvm/prebuilt/darwin-x86_64
export TOOLCHAIN=$NDK/toolchains/llvm/prebuilt/linux-x86_64
# 指定目标架构三元组
export TARGET=aarch64-linux-android
export TARGET=armv7a-linux-androideabi
export TARGET=i686-linux-android
export TARGET=x86_64-linux-android
# 指定安桌最低API
export API=21
# 指定 binutils 
export AR=$TOOLCHAIN/bin/$TARGET-ar
export AS=$TOOLCHAIN/bin/$TARGET-as
export CC=$TOOLCHAIN/bin/$TARGET$API-clang
export CXX=$TOOLCHAIN/bin/$TARGET$API-clang++
export LD=$TOOLCHAIN/bin/$TARGET-ld
export RANLIB=$TOOLCHAIN/bin/$TARGET-ranlib
export STRIP=$TOOLCHAIN/bin/$TARGET-strip
./configure --host $TARGET
make
```

对于非 AUTOCONF 的项目，有的可以用 AUTOCONF项目的方式进行替换，但有的不行，必须手动进行替换。

# luasocket 手动配置示例

这个项目比较简单，不外乎就是配置好工具链，然后编译就行，这里我们显示可能出现的问题，我们故意使用了 `armv7a-linux-androideabi` 这个架构：

```sh
git clone https://github.com/diegonehab/luasocket.git 
cd luasocket 

export TOOLCHAIN=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/darwin-x86_64
# export TOOLCHAIN=$NDK/toolchains/llvm/prebuilt/linux-x86_64
# 指定目标架构三元组
# export TARGET=aarch64-linux-android
export TARGET=armv7a-linux-androideabi
# export TARGET=i686-linux-android
# export TARGET=x86_64-linux-android
# 指定安桌最低API
export API=21
# 指定 binutils 
export AR=$TOOLCHAIN/bin/$TARGET-ar
export AS=$TOOLCHAIN/bin/$TARGET-as
export CC_linux=$TOOLCHAIN/bin/$TARGET$API-clang
export CXX=$TOOLCHAIN/bin/$TARGET$API-clang++
export LD_linux=$TOOLCHAIN/bin/$TARGET-ld
export RANLIB=$TOOLCHAIN/bin/$TARGET-ranlib
export STRIP=$TOOLCHAIN/bin/$TARGET-strip

# 这下面是和项目相关的
export PLAT=linux LUAV=5.3
export LUAINC_linux_base=/usr/local/opt/lua/include
export LUAPREFIX_linux=build CDIR_linux=lib LDIR_linux=lua

make $@
```

这将会报错：

```
ake[2]: /Users/gowa/Library/Android/sdk/ndk/21.2.6472646/toolchains/llvm/prebuilt/darwin-x86_64/bin/armv7a-linux-androideabi-ld: No such file or directory
make[2]: *** [socket-3.0-rc1.so] Error 1
make[1]: *** [linux] Error 2
make: *** [linux] Error 2
```

这就 是上面说的，对于 `armv7a-linux-androideabi` 编译器能正常识别，但是 binuitls 无法正常识别的原因：

```sh
export LD_linux=$TOOLCHAIN/bin/arm-linux-androideabi-ld
```

这样就能成功了。，没毛病。其他的 `ar, as, ranlib, strip ` 也需要这样进行修改的

# 总结

实际上通过工具链文件就指定了我们需要使用的 NDK 编译器，及相关的工具，平台库等信息。

[1]: https://developer.android.com/ndk/guides
[2]: https://developer.android.com/ndk/guides/stable_apis.html
[3]: https://www.graphviz.org/
[4]: https://developer.android.com/studio/projects/install-ndk.md
[5]: https://developer.android.com/ndk/guides/ndk-build.html
[6]: https://developer.android.com/ndk/guides/standalone_toolchain.html
[7]: https://developer.android.com/ndk/guides/cmake.html
[8]: https://developer.android.com/studio/projects/add-native-code.html
[9]: https://cmake.org/cmake/help/latest/manual/cmake-toolchains.7.html
