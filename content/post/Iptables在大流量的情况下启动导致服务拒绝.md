---
title: Iptables在大流量的情况下启动导致服务拒绝
toc: true
date: 2016-11-14 17:46:15
tags: [运维, Iptables, Linux]
categories: 
  - [Linux/Unix]
---
大多数情况下我都喜欢也习惯于iptables来进行端口限制，转发，甚至过滤等操作，作为安全加固，在流量小的情况下是可靠的，但，但面对巨大的流量的时候，就会出现意料之外的情况了。这不，我最近就遇到了。
<!--more-->
# 起因
DNS服务器平时对外提供服务，QPS大概是3万-5万，安全厂商进行扫描的时候出现安全漏洞，需要用将端口进行限制。于是我就用了一条语句增加了规则，这机器之前并未启动iptables。

	iptables -A INPUT -s 192.168.1.0/24 -p tcp -m tcp --dprot 22 -j ACCEPT
	iptables -A INPUT -p tcp -m tcp --dport 22 -j REJECT
乍一看，是正常的，确实工作得也还不错。可过了一会机器就不能登录了。连忙前往机房，查看了一下日志 `nf_conntrack: table full, dropping packet`。
在这种情况下，启动iptables是不科学的，即使设置`net.netfilter.nf_conntrack_max/net.netfilter.nf_conntrack_tcp_timeout_established`流量大了也会出现问题。
# 为什么会出现这样情况
启动iptables的时候，就会加载`nf_conntrack`模块，运行于内核，用来跟踪链接状态，iptables的`nat和state`模块就会用到其跟踪的状态。  
查看一下iptables的默认规则就会发现其中一条规则：  

	iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
这一条一般都在第一个，是为了将相关联的，通信中的状态直接放行，而不用匹配到最终规则后才用默认动作`ACCEPT`进行处理。  
但这完全没有任何意义，对于DNS服务器来说，其并不需要进行后续连接状态的跟踪，响应报文就OK了，而iptable在启动后，每个报文都会匹配一下规则，并且被`nf_conntrack`跟踪状态，但无法跟踪，则会丢包，拒绝服务。    
# 解决办法
我们应该将`DNS服务53端口出去或进来`的包都禁用跟踪，这需要在raw/PREROUTING进行处理。
	
	iptables -t raw -A PREROUTING -p udp -m udp --dport 53 -j NOTRACK
	iptables -t raw -A OUTPUT -p udp -m udp --sport 53 -j NOTRACK

同样再本地环回接口来的包我们也不应该进行跟踪处理：   

	iptables -t raw -A PREROUTING -i lo -j NOTRACK
	iptables -t raw -A OUTPUT -o lo -j NOTRACK

但是但我们进行这样的规则实质后，后续的任何依赖链接状态的规则都将不能正常工作，因为无法匹配其状态。

或者我们可以将流量的过滤放在上层的防火墙设备进行，这样就最好了。
