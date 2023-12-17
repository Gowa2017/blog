---
title: Linux的用户权限与Saved set-user-id
toc: true
date: 2016-11-19 16:51:52
tags: [Apue, Linux]
categories: 
  - [Linux/Unix]
---
Linux的程序、文件权限是一个很奇妙的设置，不进行一下深入的研究，总是会出现这样那样的疑问。特别是对s权限的疑问，估计很多人都想知道是为什么呢。
<!--more-->
# fork函数
看到 __AUPE 8.10节 fork函数__ 的时候，对于调用一次返回两次感觉好奇妙，而且之后父、子进程会共同执行更是感觉不可思议。  
执行`fork`后，父子进程继续执行fork命令后的指令，子进程是父进程的副本，有自己的 __堆、栈和数据空间__ 但与父进程共享正文段。  
由于fork后经常跟随着的是exec，所以进行堆、栈、数据空间的复制是一个不必要的操作，所以很多实现采用了 __写时复制(Copy-On-Write, COW技术)__ 。    
这些区域父子进程共享，并被内核设置为只读，只有当父、子进程试图进行修改的时候才会对那块区域做一个副本，通常是虚拟存储器的一"页"。    
子进程对父进程打开的文件描述符拥有副本。

在 __3.14节__，fcntl函数进行了定义：
```
#include <fcntl.h>
int fcntl(int fd, int cmd, .../* int arg */);
```
出错返回-1,具体返回值依赖于cmd参数。
其中提到一个CMD，`FD_GETFD FD_SETFD`所返回的文件描述符标志`FD_CLOEXEC`(当前的文件描述符标志只有这一个)。
在fork后我们经常会执行exec，若设置了FD_CLOEXEC标志，子进程对父进程的文件描述符副本，被关闭。否则，将不关闭。
默认情况下，子进程调用exec并不关闭这些文件描述符副本，除非显式地用fcntl(fd,FD_SETFL,1)进行设置。
对于目录流，POSIX.1明确要求将此标志设置为1,通常，由opendir函数调用fcntl实现。

# 与进程关联的ID
与一个进程关联的ID有6个。
`real uid/real gid`  login时从登录文件取出，显然，你用什么用户登录系统，实际用户就是谁。
`effective uid/effective gid/supplementary group IDs`
`saved set-user-ID/saved set-group-ID` 被exec函数保存的`saved set-user-ID/saved set-group-ID`。

当一个程序被加载到内存执行的时候，通常用的是exec函数中的一个。
一般情况下，`real uid`与`effective uid`是相同的，你以什么身份执行一个程序，这个进程就属于谁。
但是：当`set-user-ID`位被设置，也就是文件权限位第三位是s的时候，exec将会：把进程的`effective user id` 设置为`文件所有者ID`。
这个时候，如果不属于当前用户的文件被执行，其`effective user id = 文件所有者ID`。
同时，exec还会将此文件所有者ID进行保存，所以，才叫`saved set-user-id`
那么，当执行一个具有`set-user-id`位的程序的时候，进程的`real user id` 与 `effective user id` 是不一定相同的。

我们可以用一个小程序来验证是否属实。
这个程序打印 程序运行时候的 `real user ID`与`effective user ID`。
```
#include <stdio.h>
#include <unistd.h>
int
main(int argc, char *argv[])
{
puts("I am child process");
printf("userId=%d,euserId=%d\n",getuid(),geteuid());
return 0;
}
```
同时将其权限设置为 `-rwsr-xr-x` 中的's'就是`saved-user-Id`，当执行此程序时将以超级用户权限执行。要想验证很简单。
编译后执行他。获得输出：

>I am child process
>userId=500,euserId=0

同时，我们可以在程序中调用`int setuid (uid_t newuid)`函数来更改进程的`real user id, effective user id`
但是我们必须明白：
# user id 更改规则
1、只有root进程可以更改 `real user id , saved set-user-id`。
2、当root进程调用setuid函数时 `real user id , effective user id , saved-user-id`都被设置为newuid;
3、如果一个不具有root权限的进程试图调用此函数，会出现什么呢？

如果 newuid == real user id or newuid == saved set-user-id
那么进程的 uid = newuid。
不然，出错。
同时，任何时候，进程都可以用setuid函数将，进程UID设置为：`real user id` 或者 `saved set-user-id`;

我们可以用 getuid() geteuid() 函数获取 `real user id` 和 `effective user id`,但我们无法用函数获得 `saved set-user-id`;
