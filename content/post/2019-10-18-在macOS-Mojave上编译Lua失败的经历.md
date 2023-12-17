---
title: 在macOS-Mojave上编译Lua失败的经历
categories:
  - [macOS]
date: 2019-10-18 21:46:12
updated: 2019-10-18 21:46:12
tags: 
  - macOS
  - Lua
---
之前在 mac 上编译 skynet 都是好好的，结果升级到 10.14.6 后，编译 Lua 居然失败。后面仔细的研究了整个过程，总算解决了。原因还是不很明确。
<!--more-->

# 报错代码提示

```
ld: warning: ignoring file liblua.a, building for macOS-x86_64 but attempting to link with file built for macOS-x86_64
Undefined symbols for architecture x86_64:
  "_luaL_callmeta", referenced from:
      _msghandler in lua.o
  "_luaL_checkstack", referenced from:
      _pmain in lua.o
```

反正就是这么一串，从代码来看，是在进行 链接的时候找不到那些符号码。但是都已经编译打包到 liblua.a 里面去了，为什么还会没有呢？

注意到了那个 ignoring file ，说这个库文件被忽略了，那肯定链接不上，为什么会被忽略呢。后面的提示是平台架构不同，但是确实是架构一样的。

```
objdump -a liblua.a
In archive liblua.a:

lapi.o:     file format mach-o-x86-64
rw-r--r-- 0/0  27248 Jan  1 08:00 1970 lapi.o


lcode.o:     file format mach-o-x86-64
rw-r--r-- 0/0  17068 Jan  1 08:00 1970 lcode.o


lctype.o:     file format mach-o-x86-64
rw-r--r-- 0/0    684 Jan  1 08:00 1970 lctype.o

```

谷歌搜索了很多，都看不到解决的办法。终于在晚上看到一个 [github 上 issue 的答复](https://github.com/tpoechtrager/osxcross/issues/11#issuecomment-39827659)：



>You must use the correct ar (archiver), e.g.:

>make CXX=o64-clang++ AR=x86_64-apple-darwin1X-ar

>Otherwises it uses the system ar and won't work.

[StackOverFlow 上也有一个相似的问题](https://stackoverflow.com/questions/30948807/static-library-link-issue-with-mac-os-x-symbols-not-found-for-architecture-x8)

说是使用的 ar 命令不对（或者是说 GNU 和 macOS ar命令的行为不太一样）。隐约记得我是装了一个 binutils 的，难道被覆盖了。

于是查了一下：

```sh
which ar ranlib
/usr/local/opt/binutils/bin/ar
/usr/local/opt/binutils/bin/ranlib
```

果然，那么先把命令换了。

# 解决办法

打开 Lua 的 Makefile 文件，替换这两个命令：

```
AR=  ar rcu
RANLIB= ranlib

```

替换成：

```
AR=  /usr/bin/ar rcu
RANLIB= /usr/bin/ranlib
```

之后编译，果然就OK了。


StackOverFlow 上的解决方案是，不要使用 ar,使用  libtool 命令来生成静态库，同时也使用 macOS 自带的ranlib 命令

```sh
libtool -static -o liblua.a lapi.o lcode.o lctype.o ldebug.o ldo.o ldump.o lfunc.o lgc.o llex.o lmem.o lobject.o lopcodes.o lparser.o lstate.o lstring.o ltable.o ltm.o lundump.o lvm.o lzio.o lauxlib.o lbaselib.o lbitlib.o lcorolib.o ldblib.o liolib.o lmathlib.o loslib.o lstrlib.o ltablib.o lutf8lib.o loadlib.o linit.o
/usr/bin/ranlib liblua.a

 ld -o lua   lua.o liblua.a  -lm -lreadline
```

这样也是OK的。
# 补充

## ar 命令

这个命令是将多个 obj 文件打包成一个静态 .a 库文件。其用法类似于压缩命令啊。

## ranlib   

这个命令会将 ar 打包后的文件，里面所有的 object 文件定义的符号，生成一个索引存在里面。可以加快链接速度。 GNU 的 ranlib 命令是和 `ar -s ` 命令等价的。


猜测就是 GNU 的 ar/ranlib 命令和 macOS 上的不一致，所以才会出问题。
