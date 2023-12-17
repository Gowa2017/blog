---
title: Hooking-Linux中的共享库函数
categories:
  - [Android]
  - [Linux/Unix]
date: 2019-11-20 10:55:12
updated: 2019-11-20 10:55:12
tags: 
  - Android
  - 逆向
  - Hooking
---

函数 Hooking 指的是一系列的在运行时用来拦截和改变已存函数行为的技术。本节使用动态加载API来演示一个进行 Hooking 的办法，主要是利用了 **LD_PRELOAD** 环境变量。**LD_PRELOAD**环境变量用来指定一个首先被加载器加载的共享库。先加载我们自己的共享库就能使我们拦截函数调用，接着我们就可以使用动态加载器的API来将原始的函数绑定到一个 **函数指针**，之后继续调用这个函数指针。也就是说对原来的函数做了一个包装。
<!--more-->



# Hello World

```c
#include <stdio.h>
#include <unistd.h>
int main()
{
puts("Hello world!n");
return 0;
}
```



共享库代码：

```c
#include <stdio.h>
#include <unistd.h>
#include <dlfcn.h>
#include <string.h>

int puts(const char *message){
    int (* o_puts)(const char *message);
    int ret;
    o_puts = dlsym(RTLD_NEXT, "puts");
    o_puts("hooking");
    if (strcmp(message,"Hello world!n") == 0){
        ret = o_puts("Good Bye curel world!n");
    } else {
        ret = o_puts(message);
    }
    return ret;
}

```

这个例子中，我们要对 `puts` 进行 hook，有几点要说明一下：

- 我们定义了一个与原来的函数签名一样的 `puts` 函数。
- 通过 dysm 找到 puts 符号指向的地址，然后使用一个指针指向它。

# Compile

我们写了个自己的makefile文件：

```make
PLAT:=$(shell uname)
CFLAG=-fPIC -ldl
SO_SUFF=so
SO_OUT=puts.${SO_SUFF}
DYFMT=-shared
   ENV=LD_PRELOAD=./${SO_OUT}
ifeq (${PLAT},Darwin)
   DYFMT=-dynamiclib
   SO_SUFF=dylib
   ENV=DYLD_FORCE_FLAT_NAMESPACE=1 DYLD_INSERT_LIBRARIES=${SO_OUT} 
endif

all: so a.c
	cc a.c
	${ENV} ./a.out
so: puts.c
	cc ${CFLAG} ${DYFMT} -o puts.${SO_SUFF} $?
clean:
	rm -f a.out *.o *.dylib

```

我们make 以后可以看到，输出了

```
hooking
Good Bye curel world!n
```

就是算注入进去了。
