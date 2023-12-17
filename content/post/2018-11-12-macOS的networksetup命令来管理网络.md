---
title: macOS的networksetup命令来管理网络
categories:
  - macOS
date: 2018-11-12 22:31:12
updated: 2018-11-12 22:31:12
tags: 
  - macOS
  - networksetup
---
主要的原因是需要在电脑上同时连接内外网的时候，每次都需要在我所使用的 Wi-Fi 网卡上附加一个我们内网的 IP：172.28.20.1/16 之后再加上一个路由。但是每次重启电脑后都得重新添加。当然，我们可以以脚本的形式在登录时进行执行，或者用 automator 建立一个小程序，给用户进行 LoginItems 内弄上。但这觉得都不是根本的解决办法。
<!--more-->

# 一句话介绍
networksetup 是在系统偏好设置中对网络设定进行配置的工具。 networksetup 命令最少需要 `admin` 权限。 大多数的配置命令都需要 `root` 权限。

在任何需要密码的地方，可以用 `-` 代替，表示从标准输入读入密码。

# 概要

```
     networksetup [-listnetworkserviceorder] [-listallnetworkservices] [-listallhardwareports] [-detectnewhardware] [-getmacaddress hardwareport]
                  [-getcomputername] [-setcomputername computername] [-getinfo networkservice] [-setmanual networkservice ip subnet router]
                  [-setdhcp networkservice [clientid]] [-setbootp networkservice] [-setmanualwithdhcprouter networkservice ip]
                  [-getadditionalroutes networkservice]
                  [-setadditionalroutes networkservice [dest1 mask1 gate1] [dest2 mask2 gate2] ... [destN maskN gateN]] [-setv4off networkservice]
                  [-setv6off networkservice] [-setv6automatic networkservice] [-setv6linklocal networkservice]
                  [-setv6manual networkservice address prefixLength router] [-getv6additionalroutes networkservice]
                  [-setv6additionalroutes networkservice [dest1 prefixlength1 gate1] [dest2 prefixlength2 gate2] ... [destN prefixlengthN gateN]]
                  [-getdnsservers networkservice] [-setdnsservers networkservice dns1 [dns2] [...]] [-getsearchdomains networkservice]
                  [-setsearchdomains networkservice domain1 [domain2] [...]] [-create6to4service networkservicename] [-set6to4automatic networkservice]
                  [-set6to4manual networkservice relayAddress] [-getftpproxy networkservice]
                  [-setftpproxy networkservice domain portnumber authenticated username password] [-setftpproxystate networkservice on | off]
                  [-getwebproxy networkservice] [-setwebproxy networkservice domain portnumber authenticated username password]
                  [-setwebproxystate networkservice on | off] [-getsecurewebproxy networkservice]
                  [-setsecurewebproxy networkservice domain portnumber authenticated username password] [-setsecurewebproxystate networkservice on | off]
                  [-getstreamingproxy networkservice] [-setstreamingproxy networkservice domain portnumber authenticated username password]
                  [-setstreamingproxystate networkservice on | off] [-getgopherproxy networkservice]
                  [-setgopherproxy networkservice domain portnumber authenticated username password] [-setgopherproxystate networkservice on | off]
                  [-getsocksfirewallproxy networkservice] [-setsocksfirewallproxy networkservice domain portnumber authenticated username password]
                  [-setsocksfirewallproxystate networkservice on | off] [-getproxybypassdomains networkservice]
                  [-setproxybypassdomains networkservice domain1 [domain2] [...]] [-getproxyautodiscovery networkservice]
                  [-setproxyautodiscovery networkservice on | off] [-getpassiveftp networkservice] [-setpassiveftp networkservice on | off]
                  [-getairportnetwork device] [-setairportnetwork device network [password]] [-getairportpower device] [-setairportpower device on | off]
                  [-listpreferredwirelessnetworks hardwareport] [-addpreferredwirelessnetworkatindex hardwareport network index securitytype [password]]
                  [-removepreferredwirelessnetwork hardwareport network] [-removeallpreferredwirelessnetworks hardwareport]
                  [-getnetworkserviceenabled networkservice] [-setnetworkserviceenabled networkservice on | off]
                  [-createnetworkservice networkservicename hardwareport] [-renamenetworkservice networkservice newnetworkservicename]
                  [-duplicatenetworkservice networkservice newnetworkservicename] [-removenetworkservice networkservice]
                  [-ordernetworkservices service1 [service2] [service3] [...]] [-getMTU hardwareport] [-setMTU hardwarePort value]
                  [-listvalidMTUrange hardwareport] [-getmedia hardwareport] [-setmedia hardwareport subtype [option1] [option2] [...]]
                  [-listvalidmedia hardwareport] [-createVLAN name parentdevice tag] [-deleteVLAN name parentdevice tag] [-listVLANs]
                  [-listdevicesthatsupportVLAN] [-isBondSupported device] [-createBond name [device1] [device2] [...]] [-deleteBond bond]
                  [-addDeviceToBond device bond] [-removeDeviceFromBond device bond] [-listBonds] [-showBondStatus bond] [-listpppoeservices]
                  [-showpppoestatus name] [-createpppoeservice device name account password [pppoeName]] [-deletepppoeservice service]
                  [-setpppoeaccountname service account] [-setpppoepassword service password] [-connectpppoeservice service]
                  [-disconnectpppoeservice service] [-listlocations] [-getcurrentlocation] [-createlocation location [populate]]
                  [-deletelocation location] [-switchtolocation location] [-listalluserprofiles] [-listloginprofiles service]
                  [-enablesystemprofile service on | off] [-enableloginprofile service profile on | off] [-enableuserprofile profile on | off]
                  [-import8021xProfiles service path] [-export8021xProfiles service path yes | no] [-export8021xUserProfiles path yes | no]
                  [-export8021xLoginProfiles service path yes | no] [-export8021xSystemProfile service path yes | no]
                  [-settlsidentityonsystemprofile service path passphrase] [-settlsidentityonuserprofile profile path passphrase]
                  [-deletesystemprofile service] [-deleteloginprofile service profile] [-deleteuserprofile profile] [-version] [-help] [-printcommands]                  
```
下面是所有的 flags 列表及他们的描述：


## 网络服务

###  -listnetworkserviceorder
按照与一个连接的相关性列出网络服务及其对应的 **Port** 。含有 `*` 说明这个服务不可用。如：

```
 networksetup -listnetworkserviceorder
An asterisk (*) denotes that a network service is disabled.
(1) Wi-Fi
(Hardware Port: Wi-Fi, Device: en0)

(2) iPhone USB
(Hardware Port: iPhone USB, Device: en4)

(3) Bluetooth PAN
(Hardware Port: Bluetooth PAN, Device: en2)

(4) Thunderbolt Bridge
(Hardware Port: Thunderbolt Bridge, Device: bridge0)
```
###  -listallnetworkservices
这个只是简单的列出网络服务名称而已。

```
networksetup -listallnetworkservices  
An asterisk (*) denotes that a network service is disabled.
Wi-Fi
iPhone USB
Bluetooth PAN
Thunderbolt Bridge
```
###  -listallhardwareports

这个会列出所有硬件端口，包含对应的设备名称及地址。

```
networksetup -listallhardwareports  

Hardware Port: Wi-Fi
Device: en0
Ethernet Address: 34:36:3b:17:ac:be

Hardware Port: Bluetooth PAN
Device: en2
Ethernet Address: 34:36:3b:17:ac:bf

Hardware Port: Thunderbolt 1
Device: en1
Ethernet Address: 32:00:17:5f:00:00

Hardware Port: Thunderbolt Bridge
Device: bridge0
Ethernet Address: 32:00:17:5f:00:00

VLAN Configurations
===================
```
### -detectnewhardware

检测网络硬件，并为这个硬件建立一个默认的网络服务。

### -getmacaddress hardwareport

获取硬件接口的网卡地址。比如以太网或者是 Wi-Fi。

```
networksetup -getmacaddress 'Wi-Fi'                                                                                                                             130 ↵
Ethernet Address: 34:36:3b:17:ac:be (Hardware Port: Wi-Fi)
```

### -getcomputername

获取计算机名称

```
networksetup -getcomputername      
我的电脑的MacBook Air
```

### -setcomputername

设置计算机名称

```
networksetup -setcomputername "Angel's MacBook Air"  
```

### -getinfo netwokservice

获取网络服务的信息。**IP 地址，子网掩码，下一跳路由，硬件地址**


```
networksetup -getinfo "Wi-Fi"                                                                                                                                     
DHCP Configuration
IP address: 192.168.0.8
Subnet mask: 255.255.255.0
Router: 192.168.0.1
Client ID: 
IPv6: Automatic
IPv6 IP address: none
IPv6 Router: none
Wi-Fi ID: 34:36:3b:17:ac:be
```

### -setmanual networkservice ip subnet router

设置网络服务的 IP， 子网， 下一跳路由。

### -setdhcp networkservice [clientid]

设置使用 DHCP 自动获取IP地址，。 clientid 是可选的，可以将其设置为 *Empty* 来清空已设置的 clientid。

### -setbootp networkservice

设置网络服务使用 bootp

### -setmanualwithdhcprouter networkservice ip

手动指定一个 dhcp 池内的地址。

### -getadditionalroutes networkservice

获取附加的路由。

### -setadditionalroutes networkservice [dest1 mask1 gate1] [dest2 mask2 gate2] ... [destN maskN gateN]

设置附加的路由。

```
networksetup -setadditionalroutes "Wi-Fi" 172.230.1.1 255.255.0.0 172.28.20.1
```

### -setv{4 | 6}off networkservice

用来关闭 IPv4 或者 IPv6 协议。

### -setv6automatic networkservice

自动设置 IPv6 地址。

### -setv6linklocal networkservice

设置 IPv6 只使用本地链路地址。

### -setv6manual ip prefixlength router
设置 IPv6 地址。包括 *IP, 前缀, 及路由*

### -getv6additionalroutes networkservice

获取 IPv6 附加路由。

## -setv6additionalroutes networkservice [dest1 prefixlength1 gate1] [dest2 prefixlength2 gate2] ... [destN prefixlengthN gateN]

设置 IPv6 附加路由

### -getdnsservers networkservice

获取 DNS 服务器。

### -setdnsservers networkservice dns1 [dns2] [...]

设置 DNS 服务器。

```
networksetup -setdnsservers "Wi-Fi" 8.8.8.8 114.114.114.114
```

### -getsearchdomains networkservice

为指定的网络服务显示出域名。这个我们本机一般不会用到。

### -setsearchdomains networkservice domain1 [domain2] [...]

对网络服务指定域名。可设置多个呢。如果要清除的话，指定为 `aemptya`。

### -create6to4service -<newnetworkservicename>
建立一个 IPv6 -> IPv4 的网络服务。
### -set6to4automatic -<newnetworkservicename>
### -set6to4manual -<newnetworkservicename> -<relayaddress>

### -getftpproxy networkservice

获取 ftp 代理情况。

```
Enabled: No
Server: 
Port: 0
Authenticated Proxy Enabled: 0
```

### -setftpproxy networkservice domain portnumber authenticated username password

设置 ftp 代理信息。authenticated 可以是 [ on | off ]。如果设置为 on，那么需要输入后面的账户和密码。

### -setftpproxystate networkservice on | off

开/关 ftp 代理。

### -getwebproxy networkservice

获取 web 代理信息。

### -setwebproxy networkservice domain portnumber authenticated username password

设置 web 代理。
### -setwebproxystate networkservice on | off

开关 web 代理。

### -getsecurewebproxy networkservice
### -setsecurewebproxy networkservice domain portnumber authenticated username password
### -setsecurewebproxystate networkservice on | off
### -getstreamingproxy networkservice
### -setstreamingproxy networkservice domain portnumber authenticated username password
### -setstreamingproxystate networkservice on | off
### -getgopherproxy networkservice
### -setgopherproxy networkservice domain portnumber authenticated username password
### -setgopherproxystate networkservice on | off
### -getsocksfirewallproxy networkservice
### -setsocksfirewallproxy networkservice domain portnumber authenticated username password
### -setsocksfirewallproxystate networkservice on | off
### -getproxybypassdomains networkservice
### -setproxybypassdomains networkservice domain1 [domain2] [...]
### -getproxyautodiscovery networkservice
### -setproxyautodiscovery networkservice on | off
### -getpassiveftp networkservice

FTP 被动模式是否开启。
### -setpassiveftp networkservice {on | off}

设置被动 FTP 模式。

### -setautoproxyurl networkservice url

设置自动代理配置的url。

### -getautoproxyurl networkservice
获取上面配置的信息。

```
networksetup -getautoproxyurl "Wi-Fi"
URL: http://127.0.0.1:1089/proxy.pac
Enabled: Yes
```

### -setsocksfirewallproxystate networkservice on | off

设置是否开启 SOCKS 防火墙代理。


### -getairportnetwork hardwareport

显示当前的 Wi-Fi 网络。

```
networksetup -getairportnetwork en0                                                                                                                              
Current Wi-Fi Network: 360WiFi-8471A7-5G
```

### -setairportnetwork hardwareport network [password]

连接一个 Wi-Fi 热点的意思。

### -getairportpower hardwareport

看一下 Wi-Fi 是开还是关。

### -setairportpower hardwareport on | off

开关 Wi-Fi。

### -listpreferredwirelessnetworks hardwareport

列出首选的 Wi-Fi 网络热点信息。

```
networksetup -listpreferredwirelessnetworks en0 | head
Preferred networks on en0:
	360WiFi-8471A7
	real_602
	nmj602_5G
	nmj504
	HUAWEI-E5Mini-5FF7
	iPhone
	D-GuiYang
	360WiFi-DD0658
	Wo4G-G7DQ
```

### -addpreferredwirelessnetworkatindex hardwareport network index securitytype [password]

添加 Wi-Fi 网络信息。


### -removepreferredwirelessnetwork hardwareport network

移除 Wi-Fi 网络。


### -removeallpreferredwirelessnetworks hardwareport

移除所有的 Wi-Fi 网络。

### -getnetworkserviceenabled networkservice

查看网络服务是否开启。

### -setnetworkserviceenabled networkservice on | off

开/关一个网络服务


### -createnetworkservice networkservicename hardwareport

在硬件端口 hadrwareport 上建立网络服务 *networkservicename*。 建立后默认会开启。

### -renamenetworkservice networkservice newnetworkservicename
重命名网络服务

### -duplicatenetworkservice networkservice newnetworkservicename
复制网络服务。

### -removenetworkservice networkservice
删除网络服务

### -ordernetworkservices service1 [service2] [service3] [...]

排序网络服务。

### -getMTU hardwareport
获取 MTU 以太网 一般是1468
### -setMTU hardwarePort value
手动设置 MTU
### -listValidMTURange hardwareport
查看有效的 MTU值。

```
networksetup -listValidMTURange en0
Valid MTU Range: 1280-1500
```

### -getMedia hardwareport
获取当前媒介的设置及端口上的活跃媒介或设备。
### -setMedia hardwareport subtype [option1] [option2] [...]
指定端口的媒介类型。
### -listValidMedia hardwareport
列出可用的媒介。

### -createVLAN name parentdevice tag
在父设备 parentdevice，创建 Vlan ，标签 tag。
### -deleteVLAN name parentdevice tag
删除 Vlan
### -listVLANs
列出V Vlan

### -listdevicesthatsupportVLAN
列出支持 Vlan 的设备。

### -isBondSupported device
设备是否支持 bond。

```
networksetup -isBondSupported "Wi-Fi"
NO
```
### -createBond name [device1] [device2] [...]
建立聚合链路
### -deleteBond bond
删除链路聚合。
### -addDeviceToBond device bond
为链路添加设备
### -listBonds
列出聚合链路
### -showBondStatus bond
查看链路聚合状态

### pppoe相关
###  -listpppoeservices
###  -showpppoestatus name
###  -createpppoeservice device name account password [pppoeName]
###  -deletepppoeservice service
###  -setpppoeaccountname service account
###  -setpppoepassword service password
###  -connectpppoeservice service
###  -disconnectpppoeservice service

### -listlocations
位置列表
### -getcurrentlocation
当前位置
### -createlocation location [populate]
创建位置
### -deletelocation location
删除位置
### -switchtolocation location
切换位置。