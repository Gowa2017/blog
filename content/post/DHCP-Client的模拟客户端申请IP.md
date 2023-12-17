---
title: DHCP Client的模拟客户端申请IP
toc: true
date: 2016-12-14 11:26:46
tags: [DHCP, TCP/IP]
categories: TCP/IP
---
Dhcp Client 用来模拟用户接入，发送各种报文到DHCP系统进行申请IP。现将其使用流程进行一下介绍。
<!--more-->
# 使用前的配置。
- system.cfg 配置客户端、服务端信息。包括IP、端口、线程信息、日志记录级别等。
- create.cfg 指定构造报文的内容、模拟终端数、Option属性等属性。
- 在create.cfg 里面配置 giaddr为*192.168.0.1*，然后在管理系统为这个设备、接口、这个来源地址分配一个地址段 *192.168.0.0/24*
- option60配置为 `"2(option60_user="iptvtest9",option60_pwd="123456",option60_realm="iptv.ha",option60_enterprise_code="61")"`
- option82配置为 `2(port_type="0",slot="1",subslot="1",port="2",vpi="100",vci="200",identifier="",rack="1",frame="1",slot="1",subslot="1",port="1",xpi="4000",xci="200")`
 配置完毕即可进行执行测试并抓包。   
# 几种Option的说明
- Option60在IPoE认证流程中携带账号和密码及配置信息；MSE处理时，和MAC地址一起配合形成**MAC@Option60**格式Optioin60内容字段具体格式请参考《中国电信“我的e家”技术规范-e家终端（e8）》。
- Option82在IPoE认证流程中，用于标识用户的线路信息，DHCPServer处理时，提取Option82信息形成相应的NAS-Port-ID属性；对于Option82的规范定义参照《中国电信宽带用户接入线路标识编码格式要求》以及《中国电信PON系统用户接入端口标识编码格式要求》定义，接入线路(端口)标识信息采用Sub-option1（即AgentCircuitIDSub-option）承载。DHCPServer处理Option82时，同样按照以上规范定义，从AgentCircuitIDSub-option中获取
# 业务规则的识别
IPoE方式可以设置为[不]认证[不]计费6种方式，选择规则是根据**接口或业务类型或属地**来选择。
可以根据报文中上来的**vlan、域名、用户名前缀、侦听IP**来识别属于是什么业务（ITV、IPTV、等等）。 
然后获得每种业务的认证、计费参数，进行认证或计费。
# IPoE的认证流程
（1）用户终端发起DHCP请求，Option60携带账号和密码信息；
（2）中间途经的网络设备根据相关规范标记Option82信息；
（3）BRAS/SR收到用户请求报文，标记相应的Option82信息(如果有需要)；同时直接转请求报文中继转发给相应的DHCPServer。
（4）DHCPServer收到用户请求报文，提取请求报文中的相关信息，构造认证所需Username和Nas-Port-ID。现阶段建议Username由MAC地址和Option60信息形成，格式为：MAC@Option60；密码为任意字符串；并将Option82信息转换为NAS-Port-ID信息，送到AAA认证。建议DHCPSERVER与AAA之间的接口采用标准Radius接口。
（5）AAA对Option60信息进行解析，获取账号和密码信息，对用户进行认证；认证通过后，则向DHCPServer发回认证通过信息，并携带用户一些相关属性。MAC地址只是作为内部用户标识，不作为认证校验信息。
（6）DHCPServer根据用户不同的业务信息分配相应的地址；用户可以正常使用业务。
# CMDSH 的使用

	cd ~/tools/cmdsh
	./cmdsh 127.0.0.1 3001
## 命令列表
command key                               command description
-------------------------------------------------------------
show config debug                         显示debug配置信息
show config log                           显示log配置信息
show config auth                          显示auth配置信息
show config ipa                           显示ipa配置信息
show config protocol                      显示portocol处理配置信息
show config parse                         显示parse配置信息
show config nak                           显示nak配置信息
show config sync                          显示sync配置信息
show config cmdCtl                        显示cmdCtl配置信息
show config stat                          显示stat配置信息
show config reply                         显示reply配置信息
show data interAddr                       显示接口地址信息
show data interInfo                       显示接口信息
show data deviceInfo                      显示设备信息
show data bannedDevice                    显示黑名单设备信息
show data ipaGroup                        显示ipa组信息
show data ipaSelect                       显示ipa选择信息
show data ipPool                          显示地址池信息
show data ipPoolSelect                    显示地址池选择规则信息
show data terminateAssemble               显示终端组装信息
show data ipSegment                       显示地址段信息
show data ipoeInfo                        显示ipoe信息
show data ipoeIdentRule                   显示ipoe识别规则
show data serviceAcctInfo                 显示业务计费信息
show data serviceIdentRule                显示业务识别规则
show data nakReplyRule                    显示NAK包回复控制规则
show data option125                       显示option125信息
show data option43                        显示option43信息
show data option54                        显示option54信息
show data option56                        显示option56信息
show run recvListSize                     显示接收队列长度
show run parseListSize                    显示解析队列长度
show run replyListSize                    显示响应队列长度
show run authListSize                     显示认证队列长度
show run ipaListSize                      显示地址分配队列长度
show run authToggle                       显示认证开关
show run IPAConnection                    显示IPA连接状态
show version                              显示版本信息
config debug flag                         配置debug日志开关
config log errPktFlag                     配置错误包日志开关
config log errPktTimeLevel                配置错误包日志文件时间间隔
config log resultPktFlag                  配置结果记录日志开关
config log resultPktTimeLevel             配置结果日志文件时间间隔
config auth serverIp                      配置认证代理服务器地址
config auth serverPort                    配置认证代理服务器端口
config auth timeOut                       配置认证超时时间
config auth listMaxSize                   配置认证队列最大长度
config auth timeoutdealmode               配置认证超时处理模式
config auth timerPoolSize                 配置定时器池最大容量
config auth toggle                        配置认证开关
config auth monitorToggle                 配置认证超时检测开关
config auth monitorStart                  配置认证超时检测开始条件
config auth monitorStop                   配置认证超时检测结束条件
config auth crbQueueNum                   配置CRB队列最大长度
config ipa timeOut                        配置地址分配超时时间
config ipa threadNum                      配置地址分配处理线程数
config ipa listMaxSize                    配置地址分配队列最大长度
config ipa pingPreAlloc                   配置地址分配预分配PING开关
config ipa emergencyFlag                  配置地址分配应急处理开关
config protocol serverIp                  配置dhcp本地服务地址
config protocol serverPort                配置dhcp本地服务端口
config protocol paresThreadNum            配置解析线程数
config protocol socketRecvExtend          配置接收缓存扩大倍数
config protocol socketSendExtend          配置发送缓存扩大倍数
config protocol recvRate                  配置接收速率
config protocol recvListMaxSize           配置接收队列最大长度
config protocol lineRecvRate              配置线路接收速率
config protocol macRecvRate               配置MAC接收速率
config parse serviceListenIp              配置业务侦听地址
config parse informFlag                   配置inform包控制开关
config parse declineFlag                  配置decline包控制开关
config parse rebootFlag                   配置reboot包控制开关
config parse terminateIdentFlag           配置terminateIdent控制开关
config parse extraParseFlag               配置extra解析控制开关
config nak closeCode                      配置不回复NAK的错误码
config sync centerIp                      配置分配中心地址
config sync markIp                        配置软件服务实例地址
config sync markPort                      配置软件服务实例端口
config sync fileSaveInterval              配置数据储存文件保存时间间隔
config sync putUActionLevel               配置用户行为日志入库级别
config sync putUActionFilterCode          配置用户行为日志入库过滤错误号
config cmdctl listenPort                  配置CMD通信端口
config reply option82Flag                 配置回复option82控制开关
config stat pktStatSoftware               配置系统处理包统计开关
write                                     写数据文件
reboot                                    重启系统
help                                      show all command list
