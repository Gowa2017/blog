---
title: AIX用IPSec进行包过滤
date: 2016-10-31 13:56:55
categories: [Linux/Unix]
toc: true
tags: [Aix]
---
Linux上有iptables这样好用的包过滤工具，AIX肯定也有，只是以前没有接触过而已，最近要公司要给一批AIX服务器进行安全加固，所以就用到了相关的知识。IPSEC是一个协议，HU－UX，SunOS，AIX都有类似的机制。
此文讲述了IPSEC的开启及规则的建立。
<!--more-->
## AIX上的IPSEC
在AIX可以通过以下步骤打开IP Security
	smitty ipsec4 --> Start/Stop IP Security --> Start IP Security  [Now and After Reboot]
	**注意，此时请将 Deny All Non_Secure IP Packets                     [no]**
敲Enter

    Command: OK            stdout: yes           stderr: no
    Before command completion, additional instructions may appear below.
    ipsec_v4 Available
    Default rule for IPv4 in ODM has been changed.
    Successfully set default action to PERMIT

此时启动，默认IPv4规则已经设置，同时默认动作是 PERMIT。
这时我们可以用命令查看一下设备状态

	lsdev -C -c ipsec
	ipsec_v4 Available  IP Version 4 Security Extension
	ipsec_v6 Available  IP Version 6 Security Extension

相关管理命令为： **lsfilt mkfilt mvfilt chfilt ckfilt genfilt rmfilt**
### 查看规则
	lsfilt -v4 -O
	1|permit|0.0.0.0|0.0.0.0|0.0.0.0|0.0.0.0|no|udp|eq|4001|eq|4001|both|both|no|all packets|0|all|0|||Default Rule
	2|*** Dynamic filter placement rule for IKE tunnels ***|no
	0|permit|0.0.0.0|0.0.0.0|0.0.0.0|0.0.0.0|yes|all|any|0|any|0|both|both|no|all packets|0|all|0|||Default Rule

我们可以通过smitty来进行增加规则。

	smitty ipsec4 --> Advanced IP Security Configuration -> Configure IP Security Filter Rules -> Add an IP Security Filter Rules.
	
	* Rule Action                                        [permit]
	* IP Source Address                                  []
	* IP Source Mask                                     []
	  IP Destination Address                             []
	  IP Destination Mask                                []
	* Apply to Source Routing? (PERMIT/inbound only)     [yes]
	* Protocol                                           [all]
	* Source Port / ICMP Type Operation                  [any]
	* Source Port Number / ICMP Type                     [0]
	* Destination Port / ICMP Code Operation             [any]
	* Destination Port Number / ICMP Type                [0]
	* Routing                                            [both]
	* Direction                                          [both]
	* Log Control                                        [no]
	* Fragmentation Control                              [0]
	* Interface                                          []
	  Expiration Time  (sec)                             []
	  Pattern Type                                       [none]
	  Pattern                                            []
	  Description                                        []

### genfilt命令增加规则
genfilt与以上操作有异曲同工之效，基本命令模式

	genfilt -v 4|6 [ -n fid] [ -a D|P|I|L|E|H|S ] -s s_addr -m s_mask [-d d_addr] [ -M d_mask] [ -g Y|N ] [ -c protocol] [ -o s_opr] [ -p s_port] [ -O d_opr] [ -P d_port] [ -r R|L|B ] [ -w I|O|B ] [ -l Y|N ] [ -f Y|N|O|H ] [ -t tid] [ -i interface] [-D description] [-e expiration_time] [-x quoted_pattern] [-X pattern_filename ] [-C antivirus_filename]


-C antivirus_filename	指定抗病毒名。-C 标志意味着ClamAV病毒库的一些版本。
-D description	描述介绍。
-v 4|6	指定IP版本
-n fid	所添加ID将会被添加至第 fid 条规则之前
-a Action	D(eny) | P(ermit) | I(f) | (e)L(se) | E(ndif)。所有IF规则必须关联ENDIF规则结束。
-s s_addr	源地址
-m s_mask	源地址掩码
-d d_addr	目标地址
-M d_mask	目标地址掩码
-g Y|N	用于Permit规则，默认为Y，表示过滤规则可以使用源路由的IP包。
-c protocol	协议，默认all。有效值udp/icmp/icmpv6/tcp/tcp.ack/ospf/ipip/esp/ah/all
-o s_opr | ICMP Code Opertion	源端口或者ICMP类型 操作。有效值:lt/le/gt/ge/eq/neq/any。默认any，当-c ospf时，必须为any。
-p s_port	源端口或ICMP类型。
-O d_opr | ICMP Code Opertion	目标端口或者ICMP类型 操作。有效值:lt/le/gt/ge/eq/neq/any。默认any，当-c ospf时，必须为any。
-P d_port 	目标端口或ICMP类型
-r R|L|B	路由，默认B。指定规则是用于R(转发包)、L（发往或来自本机的包）、B（两者都使用）
-w I|O|B	默认B。指定规则应用于I（输入包）、O（输出包）、B（两者都使用）。使用代-x -X或-C 模式是使用O选项无效，使用B有效，但只检查输入包。
-l Y|N	是否记录（匹配规则的包）日志，默认N。
-f Y|N|O|H	分段控制、默认为Y（所有包）。N（未分段包）、O（只用于分段和分段头）、H（只应用于分段头和未分段）。
-t tid	指定于该规则相关的通道标识，所有匹配包都要经过此通道。不指定此项，规则只作用于非流量通道。
-i interface	指定接口卡，默认为all。
-e expiration_time	过期时间（秒）。
-x pattern	匹配模式
-X patternfile	匹配模式文件。每行一个模式
常用举例：

	genfilt -v 4 -a P -s 192.168.0.1 -m 255.255.255.0 -d 192.168.0.88 -M 255.255.255.255 [-o any] [-p 0] -O eq -P 22 [-r B] [-w B] -l Y [-f Y] [-i all] -D "permit lan host visit port 22" [-e 0]

默认情况下我们加限制22端口的话，对所有的包都会进行规则匹配。可以利用默认的设置而进行简略

	genfilt -v 4 -a P -s 192.168.0.1 -m 255.255.255.0 -d 192.168.0.88 -M 255.255.255.255 -O eq -P 22
	genfilt -v 4 -a D -O eq -P 22

### 使命令生效

	mkfilt -v 4|6 [-d] [-u] [-z P|D] [-g start|stop] -i

-u	激活rule table中的所有状态的规则。与-d 互斥
-d	停止表中规则。与-u 互斥
-z	使默认规则（最后一条）执行P或者D动作。
-g	日志启动与停止
-i	与-u一起使用，激活所有"active"状态规则。
