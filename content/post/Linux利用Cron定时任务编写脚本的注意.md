---
title: Linux利用Cron定时任务编写脚本的注意
toc: true
date: 2016-11-28 17:12:27
tags: [运维, Linux]
categories: 
  - [Linux/Unix]
---
设置了一个监控数据库的任务脚本，手动执行一切正常，但是，设置定时任务以后，总是不能获得正确的结果，而且没有设置一个输出日志，最后，经过了多次测试得出了结果。
<!--more-->
# Interactive Shell
从行为上讲，交互式即说的是启动一个shell，等待你的输入，然后给你反应这样。
交互式shell(Interactive Shell)指的不带`non-option`参数启动，或以`-i`选项启动，所有的输出输入错误都会发送到`Terminals`。`-s`选项可以用来设置`positional parameters`($1 $2)。
可以通过`$-`来判断当前是否是交互式shell。

	case "$-" in
	*i*)       echo This shell is interactive ;;
	*) echo This shell is not interactive ;;
	esac
可选的，启动脚本还会测试`$PS1`，

     if [ -z "$PS1" ]; then
             echo This shell is not interactive
     else
             echo This shell is interactive
     fi
而非交互式的就是，你启动shell执行一定的操作，但是并不与你进行交互。
例如，当我们以`sh cmdfile`执行脚本的时候，是以`non-interactive`执行的。
我们可以将上述两段代码放在一个脚本里面进行测试，看看输出是什么。
# Login Shell
我们把需要进行登录流程验证后启动的shell称做`Login Shell`，而不需要验证的就叫`non-Login Shell`，如sh启动shell执行的脚本，在X11桌面下启动的终端。
当前shell是`non-interactive`时，`-l`和`--login`选项执行会启动一个`login shell`并执行启动文件。
# 配置文件的读取
__交互式`login shell`或非交互式shell`--login`__
当`Bash`以交互式`login shell`或非交互式`shell`以`--login`选项启动的时候，首先会执行`/etc/profile`的命令，然后执行`~/.bash_profile`、`~/.bash_login`、`~/.profile`，可以用`--noprofile`选项禁止此行为，三个文件中，找到一个执行即返回。
当已登录shell退出时，执行`~/.bash_logout`文件。

	strace bash -l test.sh
的输出可以说明，`-l`选项确实会执行此一流程。
__交互式`non-login shell`__
`non-login shell`不用进行完整的登录流程认证，其首先执行`~/.bashrc`（可以用`--norc`禁止），`--rcfile FILE`选项强制shell读取`FILE`配置进行执行。常规地，`~/.bash_profile`会包含类似的行:

	if [ -f ~/.bashrc ]; then . ~/.bashrc; fi
以保证`login shell`与`non login shell`具有相同的环境变量。
我们可以试试注释掉 `~/.bash_profile`中的 上述代码，然后在`~/.bashrc`加入`export TST=/www/tst`，之后用
	echo "echo \$TST" >> tst.sh	
	bash -i tst.sh
看看发生了什么，去掉`-i`又发生了什么，我们用`sh`代替`bash`又发生了什么。
__非交互式shell__
执行以下命令的时候

     if [ -n "$BASH_ENV" ]; then . "$BASH_ENV"; fi
`PATH`变量不会被用来寻找`$BASH_ENV`文件。	
__以`sh`启动shell__
当以`sh`启动`bash`的时候，只会读取`/etc/profile`、`~/.profile`两个文件，同样可以用`--noprofile`进行禁止，当交互式执行`sh`的时候，`Bash`寻找变量`ENV`（如果有定义），并执行其值。`--rcfile`将不会起作用。读取完这些文件后，`bash`进入`POSIX`模式。
以`sh`启动非交互式shell的时候，不会读取其他启动文件。
# 变量是会继承的
因为linux是以`fork-and-exec`进行执行`shell`所以会共享环境变量。
	
# cron的环境变量
启动任务的时候，有自己的环境变量，在`/etc/crontab`下，不会读取其他启动文件，所以获得不了正确的环境变量。所以获得不了正确的结果。
同时是以`non-interactive`和`non-login-shell`执行的，所以不会执行`~/.bashrc`。
# source命令
事实上 `source`命令等于` .`，比如：
`source filename`与 `. filename`是等价的。
其作用是在当前的环境下读取并执行脚本后返回。如果`filename`不包含`/`，那么就用`PATH`变量中进行寻找命令，且不用具有`x`权限。在`POSIX`模式下，如果`PATH`中找不到，则会寻找当前目录。

# 文件内容的读取
## /etc/profile
在用户登录系统的时候，会执行`/etc/profile`脚本，其作用是此会导出一系列变量：

	export PATH USER LOGNAME MAIL HOSTNAME HISTSIZE INPUTRC
并会执行`/etc/profile.d/`目录下的脚本文件:
```
for i in /etc/profile.d/*.sh ; do
    if [ -r "$i" ]; then
        if [ "${-#*i}" != "$-" ]; then
            . $i
        else
            . $i >/dev/null 2>&1
        fi
    fi
done
```
包括的脚本为：
```
/etc/profile.d/colorls.sh
/etc/profile.d/cvs.sh
/etc/profile.d/glib2.sh
/etc/profile.d/krb5-devel.sh
/etc/profile.d/krb5-workstation.sh
/etc/profile.d/lang.sh
/etc/profile.d/less.sh
/etc/profile.d/vim.sh
/etc/profile.d/which-2.sh
```
