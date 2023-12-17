---
title: tcpdump的使用-过滤表达式
toc: true
date: 2016-12-08 16:13:42
tags: [Tcpdump, Linux, 运维, TCP/IP]
categories: [TCP/IP]
---
一直都有在用，但是远没有体会到强大，直到某一天，实在需要排除网络故障问题的时候，让我们一起来学习吧。
<!--more-->
# 基本命令格式
OS： CentOS 5.10 X86_64
- tcpdump version 3.9.4
- libpcap version 0.9.4
```
       tcpdump [ -AdDeflLnNOpqRStuUvxX ] [ -c count ]
               [ -C file_size ] [ -F file ]
               [ -i interface ] [ -m module ] [ -M secret ]
               [ -r file ] [ -s snaplen ] [ -T type ] [ -w file ]
               [ -W filecount ]
               [ -E spi@ipaddr algo:secret,...  ]
               [ -y datalinktype ] [ -Z user ]
               [ expression ]
```
## 选项说明
- **-A** 以ASCII打印每个报文。放用来抓网页。
- **-c** *count* 收到 count 个包后退出
- **-C** *file_size* 在将包数据写入文件的时候，检查文件是否大于设置的file_size。如果大了就重新开一个文件。文件名用**-w filename**指定，并加上一个从1开始的数字。file_size的单位是（1,000,000  bytes,  not 1,048,576 bytes）
- **-d** -dd -ddd 将捕捉条件以人易读的方式/C片段方式/10进制方式。一般不用
- **-D --list-interface** 以 1.eth0 这样的方式打印网卡，供 -i 调用。可能在老版本的 libpcap 上（缺少pcap_findalldevs()函数）无法支持。
- **-e** 打印出链路层的头部信息如mac地址。
- **-E** 针对 Ipsec ESP包。
- **-f** 用数字形式显示 '外部的' 互联网地址, 而不是字符形式。
- **-F** *file* 读取file内的表达式，命令行上的将失效。
- **-i** *interface* 指定监听网卡
- **-l** 以标准输出变成缓冲行。当在抓包的时候你也想进行查看数据时。`tcpdump -l | tee data` 或`tcpdump -l > data & tail -f data`会很有用  
- **-L** 打印数据链路类型，然后退出。
- **-m** 
- **-M**
- **-n** 并不将IP转换为主机名。用来避免DNS查询。
- **-nn** 不要把协议、端口转换为明细。比如：不要把53端口转换为DNS。
- **-N** 不显示完整的域名引用。应用此选项**nic.ddn.mil**将会只显示为**nic**
- **-O** 不运行代码优化器。
- **-q** 快速输出输出更少的协议信息，
- **-p** 不使网卡进入混杂模式。
- **-R** 
- **-r** *filename* 从文件读取报文。（**-w filename**建立）。标准输入是**-**。 
- **-S** 打印绝对，而不是相对，TCP队列号。
- **-s** 
- **-T** 
- **-t** 不打印时间戳。
- **-tt** 打印未格式化的时间戳。
- **-ttt** 打印上一行与这一行的时间变量值（ms）。
- **-tttt** 打印默认时间格式，同时前面会加上日期。
- **-u** 打印未解密的NFS**handles**
- **-U** **packet-bufferd**，每个包收到就存到**-w filename**指定的文件，而不是等输出缓冲区满。老版本libpcap缺少(pcap_dump_flush())函数的不支持。
- **-v -vv -vvv** 打印的信息一个比一个多，详细。
- **-w** *filenmae** 将报文存到 filename，可通过**-r**读入，标准输出是**-**。
- **-W** 与**-C**一起用，限制文件生成数量，文件号最大会从。会在序号前补0,以便更好的排序。如果数字序号已到最大，就从最开始的文件开始写入。
- **-x** 16进制形式打印每个包内容（不包括链路层头部）。这是一个链路层的包，所以被填充的部分也会被打印（上层网络的数据包小于数据帧最小长度）。
- **-xx** 包括链路层头部
- **-X** 以16进制和ASCII格式打印包内容（不包括链路层头部）。
- **-XX** 以16进制和ASCII格式打印包内容（包括链路层头部）。
- **-y** 指定链路层类型
- **--version**
## 表达式
用来过滤哪些报文需要被捕获，没有表达式则所有报文都会被捕获。
表达式一般包括一个或多个条件，每个条件经常是一个**限制符（qualifiers）**和一个**ID（NAME或NUMBER）**构成。
有三种限制符：
- **type** {host | net | port | portrange}，如果不指定，默认是`host`。例如：`host foo`、`net 128.3`、`port 80`、`portrange 6000-6008`。
- **dir** { src | dest | src or dst | src and dst} 指定报文方向，如果不指定，默认是`src or dst`。例如：`src foo`、`dst net 128.3`、`src or dst port ftp-data`。
- **proto** {ether | fddi | tr | wlan | ip | ip6 | arp | rarp | decnet | tcp | udp}，如果不指定，所有和**type**符合的协议会被捕获。例如`ether src foo`、`arp net 128.3`、`tcp port 21`、`udp portrange 7000-7005`。
> `fddi`与`ether`是同义词，指的是在指定网卡上所使用的链路层协议。

除了上面三种限制符外，还有一些其他的限制符：**gateway, broadcast, less, greater and arithmetic expressions**。
允许的格式：
- **dst host** _host_ 如果IPv4/v6报文的目标是*host*则为真。*host*可以是IP或者主机名
- **src host** _host_ 如果IPv4/v6报文的源地址是*host*则为真。*host*可以是IP或者主机名
- **host** _host_ 如果目标地址或源地址是_host_则为真。
- **ether [ src | dst | host}** _ehost_ 以太网帧[源地址 | 目的地址 | 目的地址或源地址]是 _ehost_
- **gateway** _host_ 如果报文的gateway是_host_。也就是说：ether帧的源地址或目的地址是_host_，但是IP报文的目的地址和源地址都不是_host_。
- **[ src | dst ] net** _{net mask netmask | net/len}_ 指定[源网络号 | 目的网络号 | 目的或源网络号]是 _net_
