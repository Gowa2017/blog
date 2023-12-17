---
title: SSH客户端文件配置
toc: true
date: 2017-1-6 10:40
tags: [SSH,Linux]
categories: [Linux/Unix]
---
在windows下，我们有xshell或者SecureCRT这样的利器供我们使用，但如果在macOS下的话用起来就恼火。但事实上我们可以通过配置ssh命令的行为来更加方便的管理设备。
<!--more-->
[官方文档链接](http://man.openbsd.org/ssh_config)
# 配置文件的读取
**ssh**命令会按以下的顺序读取文件获得配置的参数。

1. 命令行选项
2. ~/.ssh/config
3. /etc/ssh/ssh_config

> 对于每个参数，第一个获得的值将被使用。

配置文件中用`Host`分开每个节，每个节的设备只会应用到匹配上`Host`指定模式的主机上。比如

	ssh test
命令只会匹配配置文件中

	Host test
	...
	...
指定的属性。
由于只会使用第一个获取的值，所以要将某些特别的属性放在前面，共有的或默认的属性放在配置文件后面。
# 配置文件的格式
空行及以`#`开头的的行识别为注释，否则的话每行就有**`keyword arguments`**这样的格式。在参数含有空格的时候，可以使用`"`来包围参数。
`keywords`不区分大小写，但`arguments`区分大小写。
# 常用选项
- **Host** *pattern* 限制之后的直到下一个**`Host`**或**`Match`**之间的声明只应用于匹配*pattern*的主机。多个*pattern*用空白分割。`*`代表了所有主机默认选项。
- **HostName** 真实主机名。IP和域名都可以接受。
- **IdentityFile** 指定私钥文件位置，可指定多个，按序读取。
- **PasswordAuthentication** 是否使用密码认证。
- **Port** 连接端口，默认22
- **User** 登录用户名。

# 更多选项
- **Match** *pattern* 限制之后的直到下一个**`Host`**或**`Match`**之间的声明只应用于满足*pattern*。
- **AddKeysToAgent** *{ no | yes | ask | confirm }* 
- **AddressFamily** *{ any | inet | inet6 }*
- **BatchMode** *{ no | yes }* 是否关闭 密码 询问。对某些脚本任务中不需要密码工作很有用。
- **BindAddress** * addr * 使用本地机器上的*addr*IP进行连接
- **CanonicalDomains** 当**CanonicalizeHostname**启用的时候，会在此选项指定的域名后缀列表查找目标主机。
- **CanonicalizeFallbackLocal** 是否在规范化域名失败的时候返回一个错误。默认时**yes**，尝试使用系统解析器的查找规则来解析**unqualified  hostname**。设置为**no**的话，当**CanonicalizeHostname**启用并且目标**hostname**没有在**CanonicalDomains**内查找到的时候，**ssh**立即返回失败。
- **CanonicalizeHostname** *{ no | yes | always }* 控制是否执行严格的域名标准化。默认是**no**，不进行任何域名重写，让系统解析器进行域名寻找。如果设置为**yes**，对没有使用**ProxyCommand**的连接，**ssh**命令将会对在命令行指定的域名按照**CanonicalDomains**给定的后缀和**CanonicallizePermittedCNAMEs**规则。如果设置为**always**，对使用代理的连接也使用域名规范化。这个选项启用后，会重新读取配置文件用新的**target name**来获取匹配**Host**或**Match**节中的选项。
- **CanonicalizeMaxDots** 最多能指定的`.`数量。默认是1。
- **CanonicalizePermittedCNAMEs** 控制在进行主机名规范化的时候是否跟随**CNAMES**。规则有*source_domain_list:target_domain_list*。
- **CertificateFile** 指定证书文件位置，相应的私钥文件要单独提供。可指定多个，读取的时候按照顺序进行。
- **ChallengeResponseAuthentication** *{ no | yes }** 挑战相应认证是否开启。
- **CheckHostIP** *{ yes | no }* 默认**yes**会检查主机IP是否存在于*known_host*文件内。用以检查一个主机的key是否因DNS欺骗而改变，同时会增加IP进入*~/.ssh/known_hosts*而不考虑**StrictHostKeyChecking**的设置。
- **Cipher** 协议版本1中指定使用的算法。**blowfish, 3des(默认), des（传统的ssh实现）**支持。
- **Ciphers** 协议版本2指定使用的算法。用逗号分割多种算法。如果指定的值前面有一个`+`号，那么就会在默认的值后面进行增加而不是替换默认值。支持的算法如下:
>3des-cbc 
aes128-cbc 
aes192-cbc 
aes256-cbc 
aes128-ctr 
aes192-ctr 
aes256-ctr 
aes128-gcm@openssh.com 
aes256-gcm@openssh.com 
arcfour 
arcfour128 
arcfour256 
blowfish-cbc 
cast128-cbc 
chacha20-poly1305@openssh.com

默认的值是：
>chacha20-poly1305@openssh.com, 
aes128-ctr,aes192-ctr,aes256-ctr, 
aes128-gcm@openssh.com,aes256-gcm@openssh.com, 
aes128-cbc,aes192-cbc,aes256-cbc

- **ClearAllForwardings** *{ no | yes }* 所有在配置文件、命令行指定的本地、远程、动态端口转发都被清理。
- **Compression** *{ no | yes }* 是否启用压缩
- **CompressionLevel** 只是1(fast)到9(slow, best)，和gzip意义一样。默认等级是**6**。只对协议版本1有效。
- **ConnectionAttempts** 重试次数。
- **ConnectTimeout** 指定超时时间（秒），而不是使用tcp自己的超时机制。
- **ControlMaster** 启用在一个连接上共享多个会话。
- **ControlPath**
- **ControlPersist**
- **DynamicForward** 指定安全隧道端口，后续的远程机器会连接这个端口。参数必须是*[bind_address:]port*。默认情况下，端口与**GatewayPorts**设定的范围一致。支持**socks4, socks5**。
# 常见用法
**对不同的主机使用不同的私钥**
```
Host blog
	Hostname blog.tuisiyuan.net
	IdentityFile ~/.ssh/id_rsa_blog
	User root
Host me
	Hostname 120.55.87.63
	IdentityFile ~/.ssh/id_rsa_blog
	User root
```
