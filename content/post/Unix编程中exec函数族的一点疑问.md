---
title: Unix编程中exec函数族的一点疑问
toc: true
date: 2016-11-19 16:31:28
tags: [Apue, Linux]
categories: [Linux/Unix]
---
在观看 __AUPE8.12节 解释器文件__ 的时候，很迷惑不不解。问题不出在这几个函数，而在于看后面章节文解释器的时候发现一个很奇妙的问题。　
<!--more-->
# exec函数族的定义
```c
#include <unistd.h> 
int execl(const char *pathname, const char *arg0, 
... /* (char *)0 */ );
int execv(const char *pathname, char *const argv []);
int execle(const char *pathname, const char *arg0, ...
           /* (char *)0,  char *const envp[] */ );
int execve(const char *pathname, char *const
 	argv[], char *const envp []);
int execlp(const char *filename, const char *arg0,
 ... /* (char *)0 */ );
int execvp(const char *filename, char *const argv []);
```

第一个参数是路径名或者文件名， 后续的是一连串字符串参数或者指针数组。来研究一下文中的小程序。
# 举例
```c
#include "apue.h"
#include <sys/wait.h>

char    *env_init[] = { "USER=unknown", "PATH=/tmp", NULL };

int
main(void)
{
    pid_t   pid;

    if ((pid = fork()) < 0) {
        err_sys("fork error");
    } else if (pid == 0) {  /* specify pathname, specify environment */
        if (execle("/home/sar/bin/echoall", "echoall", "myarg1",
                "MY ARG2", (char *)0, env_init) < 0)
            err_sys("execle error");
    }

    if (waitpid(pid, NULL, 0) < 0)
        err_sys("wait error");

    if ((pid = fork()) < 0) {
        err_sys("fork error");
    } else if (pid == 0) {  /* specify filename, inherit environment */
        if (execlp("echoall", "echoall", "only 1 arg", (char *)0) < 0)
            err_sys("execlp error");
    }

    exit(0);
}
```
```c
#include "apue.h"

int
main(int argc, char *argv[])
{
    int         i;
    char        **ptr;
    extern char **environ;

    for (i = 0; i < argc; i++)      /* echo all command-line args */
        printf("argv[%d]: %s\n", i, argv[i]);

    for (ptr = environ; *ptr != 0; ptr++)   /* and all env strings */
        printf("%s\n", *ptr);

    exit(0);
}
```
对于：

```c
execle("/home/sar/bin/echoall", "echoall", "myarg1",
            "MY ARG2", (char *)0, env_init)
```

的调用，感性的判断认为，应该是将`echoall myarg1 "MY ARG2"`三个参数传给`echoall`那么，加上程序本身，应该是有四个参数，然而结果却不是如此。  
输出的结果是：

```bash
argv[0]: echoall
argv[1]: myarg1
argv[2]: MY ARG2
```

为何argv[0]会变成了传入的第二个参数呢。翻看英文版看到仔细的阅读了一下。

>Note also that we set the first argument, argv[0] in the new program, to be the filename component of the pathname. 
>Some shells set this argument to be the complete pathname. This is a convention only. 
>We can set argv[0] to any string we like.

这是把 __传入exec执行程序的第一个参数 新程序的`argv[0]`__ 设置为路径名的文件名部分。  
某些shell会把这个参数设置为完整路径名，只是为了方便。  
我们可以把`argv[0]`设置为任何值(pathname为完整的情况下)  

