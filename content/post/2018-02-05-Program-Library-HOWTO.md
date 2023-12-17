---
title: Program-Library-HOWTO
categories:
  - [Linux/Unix]
date: 2018-02-05 16:57:32
updated: 2018-02-05 16:57:32
tags:
  - Linux
  - So
---
这个文档讨论了怎么样在Linux上建立和使用程序库。包括了静态库，共享库，动态载入库。

# 介绍
这个 *HOWTO*讨论的是使用GNU的工具集来建立和使用程序库。一个 **程序库** 就是一个简单的包含了编译代码（和数据）的文件，在后面将会用来和一个程序共同工作；程序库允许程序更加模块化，重新编译更快，更容易更新。程序库可以被分为三类：**静态库， 共享库， 动态载入库（DL）**。
<!--more-->

首先讨论静态库，它会在一个程序运行前安装到一个可执行文件中。然后再讨论共享库，它会在程序启动时载入，然后在程序间共享。最好，讨论动态载入库，这可以在程序运行的任何时间载入。**DL库**并不真正是一个不同的库模式（静态和共享库都可以用作DL库）；只是因为其被程序员使用的方式不同而已。

大多数开发库的程序员都应该建立共享库，因为这允许用户单独的更新库而不影响应用程序。DL库是实用的，但是这需要更多的工作，很多程序并不需要这个特性。 相对的，静态库让升级库变得非常麻烦，所以一般不建议使用它。但是，每个库都有他们的优点，接下来我们会讨论到。使用 C++和DL库的开发者应该看一下`C++ dlopen mini-HOWTO`。

值得注意时，有些人使用术语*动态链接库（DLLs）*来指代共享库，某些使用 *DLL*来表示所有用做DL的库，某些使用DLL来表示所有这些意思。不管你用什么术语，这个HOWTO都覆盖了Linux的DLLs。

这个文档，只讨论 可执行和链接格式（ELF）的可执行文件和库，基本上所有的Linux发行版都使用这个格式。GNU 的 gcc工具集可以处理不是ELF格式的库；实际上，很多Linux发行板还在使用废弃的 *a.out*格式。然后，我们文档不讨论这个格式。


如果你要构建一个想要在很多系统上使用的应用，你应该考虑一下使用 `GNU libtool`来构建和安装库，而不是使用 直接使用Linux的工具。 GNU libtool是一个常规的库支持脚本，隐藏了使用共享库的复杂性（比如，建立和安装）。在Linux上，GNU libtools在这些工具之上构建，在本文档也有描述。为动态载入库提供一个可移植的接口，可以使用很多可移植的封装方法。GNU libtools包含了一个这样的封装器，叫做`libltdl`。可选地，你也可以使用 glib 库（不要和glibc混淆）对 模块动态载入的支持。可以在这里了解更多 glib的知识[glib](http://developer.gnome.org/doc/API/glib/glib-dynamic-loading-of-modules.html)。

# 静态库
静态库只是普通对象文件的简单集合；一般来说，静态库以`.a`结尾，使用`ar(archiver)`程序来建立。静态库并不经常使用，因为共享库的比它更有优势。但是呢，某些时候还是需要使用，首先是因为历史因素，再者，解释起来简单。

静态库允许用户不需要重新编译代码就能链接到程序中，这样就减少了重新编译的时间。不过在今天编译器已经非常快速的情况下，其实编译时间并不重要，所以这个原因并不像以前那么有用。静态库对于开发者只允许程序员链接至他们的库，但并不想提供源代码的时候非常有用。理论上，静态 ELF 库链接至可执行文件后，运行速度会比共享库或动态载入库轻微加快（1-5%），但实际上这并不是使用它的原因。

为了建立一个静态库，或者把当前目标文件加入到已有的静态库，使用下面的命令：

```bash
ar rcs my_library.a file1.o file2.o
```

建立了库后可能就想使用它。在建立可执行文件时，可以把它编译或链接过程的一部分。如果是使用`gcc`来产生可执行文件，可以使用`-l`选项来指定这个库。

要小心使用gcc时的参数顺序；`-l`是一个链接器选项，因此需要放在被编译文件的后面。这和通常的选项语法有所不同。如果你把`-l`选项放在被编译文件的前面，这会失败，并发生错误。

也可以使用链接器`ld`，加上`-l, -L`选项；然而，多数时候使用gcc会更好，因为`ld`的接口可能会改变。

# 共享库

共享库是在程序在启动时加载的库。一个共享库正确的安装好，后续启动的程序都可使用这个库。但实际上比描述的更加复杂和灵活，因为Linux使用的方式允许我们：

* 更新库，并允许程序员使用老版本的库，不用保持库的后向兼容。
* 重写指定库或库中指定函数。
* 在程序运行时就跟所有这些事情。

## 约定
为了让共享库支持所有希望的属性，很多约定和指导必须遵守。必须要明白库 名字间的不同，实际上就是`soname`和`real name`（已经他们怎么交互）。也要明白他们放在文件系统中的位置。

### 共享库名字

每个共享库有一个特殊的`soname`。`soname`有前缀`lib`，后跟`.so`，后跟一个 `.`和一个版本号（在接口改变的时候，版本号会增加）（一个特殊的例外，最底层的C库并不以　lib 开头）。一个完全引用的 `soname`包括了其所在的目录；在一个正常工作的系统中，一个全引用的 `soname`只是简单的对 库`real name`的符号链接。

每个共享库有一个`real name`，就是包含库代码的文件名字。`真实名字`在 `soname`后加上`.`，次版本号，另外一个`.`，发布号。后面的`.`和发布号是可选的。

一个次版本号和发布号通过让你知道你安装库的确切版本来支持配置控制。要注意，这些数字可能和库文档中所有的不一致，尽管这会让事情变得更简单。

另外，这里还有一个编译器在需要一个库时使用的名字，我们把它叫做`链接名(linker name)`，这个名字就是`soname` 去掉版本号。

管理共享库的关键就是分开它们的名字。程序员，当列出他们需要的库时，应该只是列出他们需要的`soname`。相反，当你建立一个共享库时，你只需要以一个特别的名字建立这个库（和更多详细的版本信息）。当安装一个库的新版本时，把它放在几个特殊的目录内，然后执行程序 `ldconfig(8)`。ldconfig会测试已存文件，然后建立 `soname`符号链接至` 真实名字`，同时设置缓存文件`/etc/ld.so.cache`。

ldconfig不会设置`linker name`；典型的是在库安装时进行设置，`linker name`只是简单的连接至最新版本的`soname`或者`real name`。建议把`linker name`链接至`soname`，因为大多数情况下在更新库的时候，我们都想要在链接时自动使用它。我询问过 **H. J. Lu**为什么 ldconfig 不会自动设置 `linker name`。他的解释基于用户可能希望用最新版本的库来运行代码，但可能需要 开发时链接至一个老的不兼容库。因此，ldconfig不会对 程序员怎么链接做任何建设，所以安装者必须手动的来指定链接至哪一个版本的库。

### 文件系统位置

共享库必须放在文件系统的某些地方。多数开源软件遵守GNU标准。GNU标准建议将所有的库默认安装在`/usr/local/lib`中（所有的命令建议放在`/usr/local/bin`中）。他们也定义了覆盖这些默认设置的约定和安装的过程。

文件分层标准（FHS）在一个发行中应该放在哪里。根据FHS，大多数库应该安装在`/usr/lib`，但是如果是启动系统所需要的应该安装在`/lib`，不是系统部分的库应该在`/usr/local/lib`。


在这两个文档中并没有真正的冲突：GNU标准建议的是源代码的开发者默认设置，FHS建议的是发行者的默认设置（会选择性的覆盖源代码默认设置，通常是通过系统的包管理系统）。实践中工作得很好：最新版本的代码会自动安装在`/usr/local`，一旦代码成熟了，包管理器就可以修改默认设置把它安装到发行版的标准位置去。 注意，如果你的库调用了只能被库调用的程序，你应该把这些程序放在`/usr/local/libexec`（某些发行版中是`/usr/libexec`）。某个Red Hat系统在默认的库搜索路径中不包含`/usr/local/lib`；看一下下面的 `/etc/ld.so.conf`的讨论。其他标准的库位置为X-windows包含`/usr/X11R6/lib`。还要注意`/lib/security`是 PAM模块使用的，但这些库经常是作为 DL库使用。

## 怎么样使用共享库

在以 GNU glibc为基础的系统上，包括所有的Linux系统，启动一个ELF的二进制可执行文件会让程序的加载器自动运行。在Linux系统上，这个加载器叫做 `/lib/ld-linux.so.X`（X是一个版本号）。这个加载器会按序加载所有其他被程序使用的共享库。

会被搜索的目录写在`/etc/ld.so.conf`中。很多 Red Hat为基础的发行版在这个文件中不包含`/usr/local/lib`。我觉得这是一个Bug，所以把`/usr/local/lib`添加到文件`/etc/ld.so.conf`文件中是在很多这样的系统上运行很多程序的一个Bug修复。

如果只是想重写某个库中的几个函数，可以在`/etc/ld.so.preload`文件中写入想要重写的库名（`.o`文件）；这些`preloading`的库会在标准设置前生效。这些预加载的文件典型的是用来紧急修复；一个发布版本通常是不包含一个这样的文件的。

在程序启动的时候搜索所有的这些目录是非常低效的，所以，使用了一个缓存安排。`ldconfig(8)`程序默认读取`/etc/ld.so.conf`文件，在动态链接目录内设置合适的符号链接（所以这会遵从标准的约定），然后写入缓存文件`/etc/ld.so.cache`，之后就可以被其他程序使用。这大大提高了访问库的速度。限制就是在一个DLL被增加，移除或DLL目录改变的时候就要运行ldconfig；运行ldconfig经常是包管理器安装一个库后的必要步骤。然后，在启动时，动态加载器就会使用`/etc/ld.so.cache`来载入其需要的库。

FreeBSD使用一个稍微不同的缓存文件名。在FreeBSD中，ELF的缓存是在`/var/run/ld-elf.so.hints`，`a.out`的缓存是在`/var/run/ld.so.hints`。这些文件也会被ldconfig更新，所以这不是很重要。


## 环境变量
很多环境变量可以控制这个过程，也有很多环境变量允许你重写这些过程。

### LD_LIBRARY_PATH

可以为当前实际的执行临时替换一个不同的库。在Linux中，环境变量`LD_LIBRARY_PATH`是一个`:`分隔的目录集合，会首先在这些目录中搜索库，然后再搜索标准位置；这在调试一个新库或者使用一个特定目标使用非标准库时非常有用。环境变量`LD_PRELOAD`列出来重写标准设置的函数的共享库，就和`/etc/ld.so.preload`做的一样。这被加载器`/lib/ld-linux.so`所实现。要注意到，`LD_LIBRARY_PATH`在多数Unix-like的系统上运行，但是并不是所有；比如，在HP-UX上这个以环境变量`SHLIB_PATH`运行，在AIX上，以环境变量`LIBPATH`运行。（语法相同，冒号分隔的目录列表）


`LD_LIBRARY_PATH`对于开发和测试是非常方便的，但不应该被普通用户一般性的使用时被安装过程所修改；查看[Why LD_LIBRARY_PATH is Bad](http://www.visi.com/~bar/ldpath.html)。但是这对开发和调试是非常有用的。如果不想设置 这个环境变量，在Linux上可以直接向加载器传递一个参数。比如，接下来的代码使用给定的`PATH`而不是环境变量的内容，然后运行给定的可执行文件。

```bash
/lib/ld-linux.so.2 --library-path PATH EXECUTABLE
```

不带参数运行`ld-linux.so`会给我们很多使用它的信息，但是不要在常规时这样使用----只是为了调试而使用。

### LD_DEBUG
GNU C载入器中另外一个使用的环境变量是`LD_DEBUG`。这将触发`dl*`函数族给出更多详细信息来表明他们在干什么。比如：

```
	export LD_DEBUG=files
	command_to_run
```
将会显示在处理过程中的文件和库，告诉你检测到的依赖关系和什么顺序载入了多个so。把LD_DEBUG设置为*bindings*显示富号绑定的信息，设置为*libs*显示库搜索路径，设置为*versions*显示版本依赖关系。


设置LD_DEBUG为*help*，然后运行一个程序的话会列出可能的选项。再次重复，LD_DEBUG不是为了常规使用，只是为了调试或者测试。

### 其他环境变量

还有很多其他控制加载过程的环境变量；他们的名字以LD_或者RTLD_开头。大多数其他变量是为了加载过程的底层调试或实现特殊的能力。大多变量都没有很好的文档；如果想要知道他们，最好的方法就是阅读 加载器的源码。（gcc的一部分）

允许用户控制动态链接库，对于采取了特殊方法的 setuid/setgid程序来说是灾难性的。因此，在GNU加载器中，如果程序是setuid/setgid的，这些变量会被忽略或不严重的限制了功能。加载器通过检查程序的身份信息来确定其是不是 setuid/setgid程序；如果 uid和 euid不同，或 gid与egid不同，加载器就认为这个程序是 setuid/setgid的，因此就会强烈的限制控制链接的能力。如果你阅读 GNU glibc 库的源码，你会看到这些；首先看一下 `elf/rtld.c, sysdeps/generic/dl-sysdep.c`。这就意味着，如果你让 uid/gid 与　euid/egid 相等，然后调用一个程序，这些变量就有完整的影响。其他 Unix-like的系统处理这样的情况有所不同，但是因为同样的原因：一个 setuid/setgid 程序不应该被环境变量过度影响。

## 建立一个共享库

建立共享库很简单。首先，使用 `gcc -fPIC ` 或 `gcc -fpic`来建立将要在库内使用目标文件。`-fPIC, -fpic`选项启动了`位置无关代码`的产生，共享库需要这样做。通过 `-Wl` gcc 选项来传递 `soname`。

```
gcc -shared -Wl, -soname, you_soname \
	-o library_name file_list library_list
```

`-Wl` 选项传递属于 链接器的选项（这个情况下是 -soname 链接器选项），在`-wl`后的冒号不是一个错误。

下面是一个例子，建立两个目标文件*a.o, h.o*，然后建立一个包含这两个文件的共享库。注意这个编译包含了调试信息`-g`，还会产生警告信息`-Wall`，对于共享库来说不是必须的，但是建议这样做。编译通过 `-c`来产生目标文件，同时包含需要的`-fPIC`选项：

```

gcc -fPIC -g -c -Wall a.c
gcc -fPIC -g -c -Wall b.c
gcc -shared -Wl,-soname,libmystuff.so.1 \
    -o libmystuff.so.1.0.1 a.o b.o -lc
```

有几点值得注意的地方：

* 不要**strip**得到的库，不是必要的话，不要加上编译选项`-fomit-frame-pointer`。生成的库会工作，但是这些动作会让调试者很难使用。
* 使用`-fPIC, -fpic`来产生代码。是否使用这两个标志根据目标来定。`-fPIC`总是会工作，但是会比`-fpic`产生更多的代码（-fPIC是大写的，就会产生更多的代码，这样来记忆）。`-fpic`会产生更少更快的代码，但是会产生平台相关的限制，比如全局可见符号数或代码的大小。链接器会在你建立一个共享库的时候告诉你是否合适。如果有疑问，使用`-fPIC`，总是没有错的。
* 某些情况下，调用 gcc来产生目标文件也需要包含选项`-Wl, -export-dynamic`。通常情况下， 动态符号表只包含被动态目标使用的符号。这个选项（在建立一个ELF文件时）把所有的符号都添加到动态符号表中（查看 ld(1)获取更多信息）。当有`反向依赖`的时候需要使用这个选项，比如，某程序要调用含有未解析的符号的DL库，而这些符号又必须在程序内定义时。为了让`反向依赖`工作，程序必须让它的符号动态可见。如果只是在Linux系统工作，可以用`-rdynamic`来替代`-Wl, -export-dynamic`，但是在ELF文档中说明，`-rdynamic`标志在非Linux系统上的gcc中并不总是能工作。

在开发着，修改一个被其他程序使用的库有潜在的问题————而且你也不需要其他程序使用开发版的库，只有特定的程序你想要进行测试。一个可能会用的链接选项就是 ld 的`rpath`，这选项指定了特定程序编译时的运行时库搜索路径。在GCC中，可以用下面的形式指定 `rpath`选项：

```
-wl,-rpath,${DEFAULT_LIB_INSTALL_PATH}
```

如果在构建库客户程序时使用这个选项，就不用因为要保证与LD_LIBRARY_PATH不相冲突而烦恼，或者需要用一些其他技术来隐藏这个库。

## 安装和使用共享库

在建立了一个库后，你可能会想要安装它。简单的方式就是复制到一个 标准的目录内然后运行 ldconfig(8)。

首先，需要建立这个共享库。 然后，设置一下必要的符号链接，实际上就是一个从 soname 到 real name的链接。最简单的方式就是：

```
ldconfig -n directory_with_shared_libraries
```

最后，在编译程序的时候，需要告诉编译器要使用的静态或者共享库。使用`-l, -L`选项。

如果不能或者不想把库安装在标准位置，就需要改变方式了。把库放在某个地方，然后告诉程序足够的信息让它能够找到这个库，有好几种方式可以做到这个。较简单的情况下可以使用gcc的`-L`标志。如果只有特定程序使用这个库的话，可以使用`莱〔rpath`方式。也可以使用环境变量来控制。实际上，可以设置LD_LIBRARY_PATH。如果使用bash，可以这样：

```
LD_LIBRARY_PATH=.:$LD_LIBRARY_PATH my_program
```

如果只是为了重写几个函数，可以通过建立一些重写目标文件然后设置LD_RELOAD。

# 实际操作
使用的环境是CentOS 7，根据上文描述，查看了一下 `/etc/ld.so.conf`文件，确实没有包含`/usr/local/lib`目录，我们来加上它，后面我们自己的库就放在这个地方了。

```
echo '/usr/local/lib' >> /etc/ld.so.conf
```

## 代码 
我们有两个源文件 `a.c, b.c`分别只是实现了简单的两个函数：

```c
// a.c

#include  <stdio.h>

int
a() {
	printf("hello, i'm in function a");
	return 0;
}
```

```c
// b.c

#include  <stdio.h>

int
b() {
	printf("hello, i'm in function b");
	return 0;
}
```

## 共享库生成

```
gcc -g -Wall -fPIC -c a.c
gcc -g -Wall -fPIC -c b.c
gcc -g -shared -Wl,-soname,libgowa.so.1 -o libgowa.so.1.0.1 a.o b.o
```

## 三个名字的链接
接下来我们把生成的库`libgowa.so.1.0.1`复制到 `/usr/local/lib`下。

	cp libgowa.so.1.0.1 /usr/local/lib

建立 soname到 real name的符号链接：

	ln -s libgowa.so.1.0.1 libgowa.so.1

建立 linker name到 soname的 链接

	ln -s libgowa.so.1 libgowa.so

运行 ldconfig:

	ldconfig -v

看一下我们当前`/usr/local/lib`目录的样子：

```
ll /usr/local/lib
lrwxrwxrwx 1 root root    12 2月   5 21:33 libgowa.so -> libgowa.so.1
lrwxrwxrwx 1 root root    16 2月   5 21:24 libgowa.so.1 -> libgowa.so.1.0.1
-rwxr-xr-x 1 root root  9360 2月   5 21:53 libgowa.so.1.0.1
```

## 使用我们的库

编写如下代码：

```c
// c.c
#include <stdio.h>

int
main() {
	a();
	b();
	printf("hello, i'm in function main");
	return 0;
}
```

编译：

	gcc -g -Wall c.c -lgowa

接下来我们执行编译出的文件`a.out`:

```
$ ./a.out
hello, i'm in function ahello, i'm in function bhello, i'm in function main
```
看起来所有的输出都到了一行上，我觉得我们需要修改一下这个库函数才行。

## 修改库
只是简单的在库函数的输出后面加上换行符。

```c
// a.c

#include  <stdio.h>

int
a() {
	printf("hello, i'm in function a\n");
	return 0;
}
```

```c
// b.c

#include  <stdio.h>

int
b() {
	printf("hello, i'm in function b\n");
	return 0;
}
```

我们来生成我们的第二个版本的库：

```
gcc -g -Wall -fPIC -c a.c
gcc -g -Wall -fPIC -c b.c
gcc -g -shared -Wl,-soname,libgowa.so.1 -o libgowa.so.1.0.2 a.o b.o
```

然后把这个新版本的库放到`/usr/local/lib`中，就一下我们当前的目录是什么样：

```
	cp libgowa.so.1.0.2 /usr/local/lib
	ll /usr/local/lib
	lrwxrwxrwx 1 root root    12 2月   5 21:33 libgowa.so -> libgowa.so.1
	lrwxrwxrwx 1 root root    16 2月   5 21:24 libgowa.so.1 -> libgowa.so.1.0.1
	-rwxr-xr-x 1 root root  9360 2月   5 21:53 libgowa.so.1.0.1
	-rwxr-xr-x 1 root root  9360 2月   5 21:57 libgowa.so.1.0.2
```

接下来我们执行一下ldconfig:

	./ldconfig -v

看一下`/usr/local/lib`目录变成什么样了：

```
	ll /usr/local/lib
	lrwxrwxrwx 1 root root    12 2月   5 21:33 libgowa.so -> libgowa.so.1
	lrwxrwxrwx 1 root root    16 2月   5 21:59 libgowa.so.1 -> libgowa.so.1.0.2
	-rwxr-xr-x 1 root root  9360 2月   5 21:53 libgowa.so.1.0.1
	-rwxr-xr-x 1 root root  9360 2月   5 21:57 libgowa.so.1.0.2
```

ldconfig 自动的把我们的 soname libgowa.so.1 链接到了新的版本 libgowa.so.1.0.2


然后执行一下我们先前的程序 `a.out`

```
	./a.out
	hello, i'm in function a
	hello, i'm in function b
	hello, i'm in function main
```

输出果然发生了变化。
