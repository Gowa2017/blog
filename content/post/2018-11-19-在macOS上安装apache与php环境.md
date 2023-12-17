---
title: 在macOS上安装apache与php环境
categories:
  - macOS
date: 2018-11-19 13:41:38
updated: 2018-11-19 13:41:38
tags: 
  - macOS
  - php
  - apache
---
为了测试而用。 macOS High Sierra 已经预装了 php7 我们只需要把他进行启用就行了。

# 修改配置文件

```sh
sudo vim /etc/apache2/httpd.conf
```

去掉这一行前的注释：

```
#LoadModule php7_module libexec/apache2/libphp7.so
```

# 启动命令
<!--more-->

```sh
sudo apachectl restart
```

完整版：

```sh
Usage: /usr/sbin/httpd [-D name] [-d directory] [-f file]
                       [-C "directive"] [-c "directive"]
                       [-k start|restart|graceful|graceful-stop|stop]
                       [-v] [-V] [-h] [-l] [-L] [-t] [-T] [-S] [-X]
Options:
  -D name            : define a name for use in <IfDefine name> directives
  -d directory       : specify an alternate initial ServerRoot
  -f file            : specify an alternate ServerConfigFile
  -C "directive"     : process directive before reading config files
  -c "directive"     : process directive after reading config files
  -e level           : show startup errors of level (see LogLevel)
  -E file            : log startup errors to file
  -v                 : show version number
  -V                 : show compile settings
  -h                 : list available command line options (this page)
  -l                 : list compiled in modules
  -L                 : list available configuration directives
  -t -D DUMP_VHOSTS  : show parsed vhost settings
  -t -D DUMP_RUN_CFG : show parsed run settings
  -S                 : a synonym for -t -D DUMP_VHOSTS -D DUMP_RUN_CFG
  -t -D DUMP_MODULES : show all loaded modules 
  -M                 : a synonym for -t -D DUMP_MODULES
  -t -D DUMP_INCLUDES: show all included configuration files
  -t                 : run syntax check for config files
  -T                 : start without DocumentRoot(s) check
  -X                 : debug mode (only one worker, do not detach)
```

# 更改网站目录

打开 `/etc/apache2/httpd.conf` 文件，然后修改 

```
DocumentRoot "/Library/WebServer/Documents"
<Directory "/Library/WebServer/Documents">
```

把这个替换为我们自己的目录：

```
DocumentRoot "/usr/local/var/www"
<Directory "usr/local/var/www">
```

在我们的目录下建立新文件 index.php

```
<?
phpinfo();
?>
```

# 重启

```
sudo apachectl restart
```
