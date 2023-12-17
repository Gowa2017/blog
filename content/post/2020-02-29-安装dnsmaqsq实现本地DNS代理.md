---
title: 安装dnsmaqsq实现本地DNS代理
categories:
  - macOS
date: 2020-02-29 21:42:50
updated: 2020-02-29 21:42:50
tags: 
  - macOS
  - Dnsmasq
---

事实上这个本来没有什么说的，不过是我在安装的过程中，居然一直启动不起来，可知最后的原因如此的简单。安装这个初衷，实际是为了解决对于某些域名的解析，想要解析到指定的IP地址，而或者是避免 DNS 被污染的问题。

<!--more-->

# 安装

用 brew 来安装 dnsmasq 实际上非常的简单：

```sh
brew update
brew install dnsmasq
```

因为我们默认的情况下使用的是`53`端口，因此这是一个特权端口，我们不能用普通的权限启动，只能用 `root` 用户来启动。

```sh
sudo brew serivces start dnsmasq
```

在此之后我们就可以看一下我们的服务是否已经启动：

```sh
brew services list
Name     Status  User Plist
dnsmasq  started root /Library/LaunchDaemons/homebrew.mxcl.dnsmasq.plist
```

但我想要看一下 53 端口是否已经监听却是不行的

```sh
lsof -i udp -P | grep :53
```

我将服务停止后，用命令

```sh
sudo dnsmasq --test

dnsmasq: failed to open pidfile /usr/local/var/run/dnsmasq/dnsmasq.pid: No such file or directory
```

我们手动建立这个目录就行了：

```sh
sudo mkdir /usr/local/var/run/dnsmasq
```



# 配置

配置并不复杂，只不过我们需要先理解一下几个原理：

1. 收到 DNS 的查询请求后，dnsmasq 会首先读取 ``etc/hosts` 下的解析记录，如果找不到对应的记录，那么就从 **上游服务器(upstream server)** 进行相关的查询，上游服务器，默认是从 `/etc/resolv.conf` 文件内进行读取的。

所以我们有几个比较重要的选项进行配置：

- resolv-file：指定一下从哪里获取上游服务器，默认是 `/etc/resolv.conf`。
- no-resolv：不要读取 `/etc/resolv.conf` 文件
- port：监听端口，默认是 `53`，如果我们不启用 `53` 端口，那么就需要做一个数据转发，将 `53` 端口的数据都转发到我们指定的端口。
- server：定义一个上游服务器，可以多个。
- no-hosts：不要读取 `/etc/hosts` 文件
- conf-dir：定义一个配置文件目录。比如我们可以将相关的 配置文件放在一起，然后 包含进来
- address：定义一个本地进行的解析。

先建立配置文件目录：

```sh
mkdir -p /usr/local/etc/dnsmasq.d
touch /usr/local/etc/dnsmasq.d/development.conf
echo "address=/.dev/127.0.0.1" >> /usr/local/etc/dnsmasq.d/development.conf
```

我们将在 `/usr/local/etc/dnsmasq.d/development.conf` 中将我们的所有开发时用到的域名进行解析到本地

我的配置：

```conf
conf-dir=/usr/local/etc/dnsmasq.d,*.conf
port=53
no-resolv
server=8.8.8.8
server=8.8.4.4
```

重启服务后，我们使用  `dig` 命令应该能看到相关的解析：

```sh
dig nm.dev @127.0.0.1
```



我的配置方式是将所有的解析都丢到 dnsmasq 里面去，这个我们只需要在

**系统偏好设置->网络->高级->DNS** 处，只指定一个 DNS 服务器 `127.0.0.1` 就行了，实际上这个操作，也会改变 `/etc/resolv.conf` 文件中的 **nameserver** 配置。

# 不使用标准端口 53

关于这个的操作，[这篇文章有说明](https://www.stevenrombauts.be/2019/06/restart-dnsmasq-without-sudo/)