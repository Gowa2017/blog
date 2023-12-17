---
title: PPPoe-以太网上的点对点传输协议
date: 2016-10-31 14:24:42
tags: [PPPoE, TCP/IP]
categories: 
  - TCP/IP
toc: true
---
PPPoE（英语：Point-to-Point Protocol Over Ethernet），以太网上的点对点协议，是将点对点协议（PPP）封装在以太网（Ethernet）框架中的一种网络隧道协议。由于协议中集成PPP协议，所以实现出传统以太网不能提供的身份验证、加密以及压缩等功能，也可用于缆线调制解调器（cable modem）和数字用户线路（DSL）等以以太网协议向用户提供接入服务的协议体系。
本质上，它是一个允许在以太网 广播域中的两个以太网接口间创建点对点隧道的协议。
<!--more-->

## 协议概述
PPPoE分为两个阶段，即Discovery（地址发现）阶段和PPP会话阶段。当某个主机希望发起一个PPPoE会话时，它必须首先执行Discovery来确定对方的以太网MAC地址并建立起一个PPPoE会话标识符SESSION_ID(Access Concentrator生成)。虽然PPP定义的是端到端的对等关系，Discovery却是天生的一种客户端-服务器关系。在Discovery的过程中,主机(作为客户端)发现某个访问集中器（Access Concentrator，作为服务器），根据网络的拓扑结构，可能主机能够跟不止一个的访问集中器通信 。Discovery阶段允许主机发现所有的访问集中器并从中选择一个。当Discovery阶段成功完成之后，主机和访问集中器两者都具备了用于在以太网上建立点到点连接所需的所有信息。
Discovery阶段保持无状态（stateless）直到建立起一个PPP会话。一旦PPP会话建立，主机和访问集中器两者都必须为一个PPP虚拟接口分配资源。



# PPP 协议

PPP 是 点对点协议（Point-to-Point Procotol）的简称，所谓的 **点对点** 是指在一个**串行链路**只有两个端点，从一个端点，发送过去的数据，必然被另外一个端点给接收到，然后在其上传输 **IP数据报**。实际上他不是单个协议，而是一多个协议的集合。它支持建立的基本方法--**链路控制协议（LCP，Link Control Protocol）**，和一系列 **NCP（网络控制协议，Net Control Procotol）**协议，用来在基本链路建立后，对各种上层协议建立网络层链路。



LCP 主要用来维护底层的双方通信路径，因此，PPP 操作只需要关注链路的两端，而不需要像 以太网和 Wi-Fi 的 MAC 层协议那样要处理共享资源的访问。一个 LCP 的操作过程大体如下：

这里我们方便我们描述 PPPoE 写成了 C/S 形式，实际上 PPP 的端点间是对等的

```mermaid
sequenceDiagram
participant C as Client
participant S as Server

C ->> S:配置请求
S -->> C:配置确认
note left of C: 链路建立
S ->> C:配置请求
C -->> S:配置确认
note right of S: 认证(CHAP)
S ->> C: 认证请求
C -->> S:认证材料
note right of S: AAAServer 认证
S -->> C: 认证成功
note left of C: 网络建立
note over C, S: 数据交换
C ->> S: 终止请求
S -->> C: 终止确认
```

# PPPoE

PPPoE 则是为了在 以太网链路上进行传输 PPP 报文而设计的，我们知道 PPP 只能在两个端点间传输数据，所以 关键再于  PPPoE 的  PADI 报文，其会向链路层的广播地址发送 一个报文，所有一台网中的设备都有可能收到这个报文，然后对此报文进行响应。

当对此报文响应的时候，确认双方的链路层地址后，就可以以点对点的形式进行传输了。



```mermaid
sequenceDiagram
participant C as Cient
participant S as Server

C ->> S: PADI[0x09]（广播）
S -->> C: PADO[0x07]（单播）
C ->> S: PADR[0x19]（单播）
S -->> C: PADS[0x65]（单播）
C -->> S: PADT[0xA7]（单播）
```



# 典型组网

```dot
digraph{
rankdir =LR
C[label=Client]
N[label=NAS]
AAA[label="AAA服务器"]

C -> N[label=PPPoE]
N -> AAA[label=Radius]
}
```

## 报文格式

### 以太网帧格式(更多介绍请查看RFC894/tcp/ip协议详解:卷1 第2章)
![帧格式](/res/pppoe-frame.png)
ETHER_TYPE设置为0x8863(Discovery阶段)或者0x8864(PPP会话阶段)
### payloadheader
![payload头部](/res/pppoe-payloadhead.png)
### payload内容
![payload内容](/res/pppoe-payloadcontent.png)
包含0个或多个tag，每个tag是一个tlv(type-length-value)三元组

## 发现阶段(discovery stage)
发现阶段分为四步:PADI、PADO、PADR、PADS。
当HOST收到PADS后，那么HOST与AC(access concentrator)间的点对点关系就已建立，进入了会话阶段(session stage)。
### PPPoE Active Discovery Initiation数据包(PADI)
主机发送DESTINATION_ADDR 为广播地址的PADI数据包，CODE域设置为0x09,SESSION_ID域必须设置为0x0000。
PADI数据包必须包含且仅包含一个TAG_TYPE为Service-Name的TAG，以表明主机请求的服务，以及任意数目的其它类型的TAG。整个PADI数据包（包括PPPoE头部）不允许超过1484个字节，以留足空间让中继代理（向数据包中）增加类型为Relay-Session-Id的TAG。
### The PPPoE Active Discovery Offer 数据包(PADO)
   如果访问集中器能够为收到的PADI请求提供服务，它将通过发送一个PADO数据包来做出应答。DESTINATION_ADDR为发送PADI的主机的单播地址，CODE域为0x07,SESSION_ID域必须设置为0x0000。 
   PADO数据包必须包含一个类型为AC-Name的TAG（包含了访问集中器的名字），与PADI中相同的Service-Name，以及任意数目的类型为Service-Name的TAG表明访问集中器提供的其它服务。如果访问集中器不能为PADI提供服务，则不允许用PADO作响应。
### The PPPoE Active Discovery Request 数据包(PADR)
   由于PADI是广播的,主机可能收到不止一个PADO,它将审查接收到的所有PADO并从中选择一个。可以根据其中的AC-Name或PADO所提供的服务来作出选择。然后主机向选中的访问集中器发送一个PADR数据包。其中，DESTINATION_ADDR域设置为发送PADO的访问集中器的单播地址，CODE域设置为0x19，SESSION_ID必须设置为0x0000。
   PADR必须包含且仅包含一个TAG_TYPE为Service-Name的TAG，表明主机请求的服务，以及任意数目其他类型的TAG。
### The PPPoE Active Discovery Session-confirmation 数据包(PADS)
   当访问集中器收到一个PADR数据包，它就准备开始一个PPP会话。它为PPPoE会话创建一个唯一的SESSION_ID并用一个PADS数据包来给主机作出响应。DESTINATION_ADDR域为发送PADR数据包的主机的单播以太网地址，CODE域设置为0x65,SESSION_ID必须设置为所创建好的PPPoE会话标识符。
   PADS数据包包含且仅包含一个TAG_TYPE为Service-Name的TAG，表明访问集中器已经接受的该PPPoE会话的服务类型，以及任意数目的其他类型的TAG。
   如果访问集中器不喜欢PADR中的Service-Name,那么它必须用一个带有类型为Service-Name-Error的TAG(以及任意数目的其它TAG类型)的PADS来作出应答。这种情况下，SESSION_ID必须设置为0x0000。
### The PPPoE Active Discovery Terminate数据包(PADT)
   这种数据包可以在会话建立以后的任意时刻发送，表明PPPoE会话已经终止。它可以由主机或访问集中器发送，DESTINATION_ADDR域为单播以太网地址，CODE域设置为0xa7,SESSION_ID必须表明终止的会话，这种数据包不需要任何TAG。
   当收到PADT以后，就不允许再使用该会话发送PPP流量了。在发送或接收到PADT后，即使是常规的PPP结束数据包也不允许发送。PPP通信双方应该使用PPP协议自身来结束PPPoE会话，但在无法使用PPP时可以使用PADT。
## 会话阶段(session stage)
一旦PPPoE会话开始，PPP数据就像其它PPP封装一样发送。所有的以太网数据包都是单播的。ETHER_TYPE域设置为0x8864。PPPoE的CODE必须设置为0x00。PPPoE会话的SESSION_ID不允许发生改变，必须是Discovery阶段所指定的值。PPPoE的payload包含一个PPP帧，帧始于PPP Protocol-ID。
## TAG_TYPE和TAG_VALUE
0x0000 End-Of-List
      该TAG表明表中没有其它TAG了。该TAG的TAG_LENGTH必须总是0。不要求使用该标签，存在是为了向后兼容。
   *0x0101 Service-Name*
      该TAG表明后面紧跟的是服务的名称。TAG_VALUE是不以NULL结束的UTF-8字符串。当TAG_LENGTH为0时，该TAG用于表明接受任何服务。使用Service-Name标签的例子是表明ISP(Internet服务提供商)或者一类服务或者服务的质量。
   *0x0102 AC-Name*
      该TAG表明后面紧跟的字符串唯一地表示了某个特定的访问集中器。它可以是商标、型号以及序列号等信息的集合，或者该访问集中器MAC地址的一个简单的UTF-8表示。它不以NULL来结束。
   *0x0103 Host-Uniq*
      该TAG由主机用于把访问集中器的响应（PADO或者PADS）与主机的某个唯一特定的请求联系起来。TAG_VALUE是主机选择的长度和值为任意的二进制数据。它不能由访问集中器解释。主机可以在PADI或者PADR中包含一个Host-Uniq标签。如果访问集中器收到了该标签，它必须在对应的PADO或者PADS中不加改变的包含该标签。
   *0x0104 AC-Cookie*
      该TAG由访问集中器用于防止拒绝服务攻击（见“安全方面的考虑”）。访问集中器可以在PADO数据包中包含该TAG。如果主机收到了该标签，它必须在接下来的PADR中不加改变的包含该标签。TAG_VALUE I是长度和值任意的二进制数据，不能由主机解释。
   *0x0105 Vendor-Specific*
      该TAG用来传送厂商自定义的信息。TAG_VALUE的头4个字节包含了厂商的识别码 ，其余字节尚未定义。厂商识别码的高字节为0，低3个字节为网络字节序的厂商的SMI网络管理专用企业码，如“定义值RFC”（参考文献[4]）中定义的那样。
      不推荐使用该TAG。为了确保互操作性，实现可以悄悄的忽略Vendor-Specific TAG。
   *0x0110 Relay-Session-Id*
      该TAG可由中继流量的中间代理加入到Discovery数据包中。TAG_VALUE对主机和访问集中器都是晦涩难懂的（paque）。如果主机或访问集中器收到该TAG，则它们必须在所有的Discovery数据包中包含该TAG以作为响应。所有的PADI数据包必须保证足够空间来加入TAG_VALUE长度为12字节的Relay-Session-Id标签。
      如果Discovery数据包中已经包含一个Relay-Session-Id标签，则不允许再加入该标签。这种情况下，中间代理应该使用该现有的Relay-Session-Id标签。如果它不能使用现有的标签，或者没有足够空间来增加一个Relay- Session-Id标签,那么它应该向发送者返回一个Generic-Error标签。
   *0x0201 Service-Name-Error*
      该TAG(典型的有一个长度为零的数据部分)表明了由于某种原因，没有理睬所请求的Service-Name。如果有数据部分,并且数据部分的头一个字节非0，那么它必须是一个      可打印的UTF-8字符串，解释请求被拒绝的原因。该字符串可以不以NULL结束。
   *0x0202 AC-System-Error*
      该TAG表明了访问集中器在处理主机请求时出现了某个错误。(例如没有足够资源来创建一个虚拟电路。PADS数据包中可以包含该标签。
      如果有数据，并且数据的第一个字节不为0，那么（数据）必须是一个可打印的UTF-8 字符串，该字符串解释了错误的性质。该字符串可以不以NULL结束。
   *0x0203 Generic-Error*
      该TAG表明发生了一个错误。当发生一个不可恢复的错误并且没有其它合适的TAG时，它可被加到PADO, PADR或PADS数据包中。如果出现数据部分，那么数据必须是一个UTF-8字符串，解释错误的性质。该字符串不允许以NULL结束。
## 数据包例子
![例子](/res/pppoe-example.png)
