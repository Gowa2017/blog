---
title: 'Linux下的系统监控命令sar'
toc: true
date: 2016-12-01 17:06:28
tags: [运维, Linux]
categories: 
  - [Linux/Unix]
---
运维人员常用的命令如**sar, vmstat, iostat**等，各有千秋与侧中，不过**sar**是比较全的查看系统性能的命令了。对于CPU、磁盘、内存、IO的性能都能进行比较完整的监控并生成报表。对于自动化运维监控具有统一化的作用。
<!--more-->
# 简单介绍
作者将代码开源在 [sar官方地址](http://sebastien.godard.pagesperso-orange.fr/)。其实是一个系统状态工具集的集合。
**sar:** 收集、报告或者保存系统活动信息。其基本命令形式如下：
```
sar [options] [-A] [-o filename] [interval[count]]
```
**sar**命令将操作系统中选定的累计活动计数器的内容写到标准输出。基于 *count* 和 *interval* 参数的值，记帐系统按指定次数(count)，以指定的时间间隔（interval，以秒为单位）写入信息。
收集的数据也可以保存在由**-o filename** 标志指定的文件中。如果**filename**省略，那么就输出到每日的记录文件**/var/log/sa/sadd**。**dd（01到31)**代表某月的某一天。默认情况下数据是按天记录在里面的。
可能你这个时候就会问了，那么，我平时都没有执行这个命令的时候，它的记录是怎么产生的呢，这就对了。
其实**sar**不止它一个程序呢，还包括下面这些程序：
- **sar** 收集、报告或存储信息（CPU、内存、磁盘、中断、网卡、TTY、内核表等等）
- **sadc** 系统数据收集器，给**sar**做后台服务。
- **sa1** 收记并存储二进制数据到每天的文件。这是设计来给**cron**执行一个**sadc**的前台程序。
- **sa2** 生成总结报表。
- **sadf** 以多种格式显示数据（CSV, XML, JSON, etc.），还可以用来生成SVG（Scalable Vector Graphics）图表，
# 先理解一些概念
## 虚拟内存(Virtual Memory)
# 选项
多的不说了，更多的介绍可以在你的系统上执行`info sar`命令查看。
- **-A**	这选项等于 *--bBdFHqrRSvwWy -I SUM -I XALL -m ALL -n ALL -r ALL -u ALL -P ALL*
- **-b**	报告**I/O**和传输状态。
	tps	每秒传输到物理设备的。一次传输就是一个I/O请求。多次逻辑的请求可以合并成一个**物理I/O**请求，每次传输的大小是不定的。
	rtps	每秒向物理设备的读请求数。
	wtps	每秒向物理设备的写请求数。
	bread/s	每秒从设备读的**block**数。
	bwrtn/s	每秒从设备写的**block**数。
- **-B**	报告页状态，下面是会显示的值。
    pgpgin/s	每秒从磁盘调入的内存页(KB)。
    pgpgout/s	每秒从调出到磁盘的内存页(KB)。
    fault/s	系统每秒产生的缺页(major+minor)。因为某些缺页不会产生**I/O**，所以这并不是一个对产生IO缺页的统计。
    majflt/s	主要页错误。
- **-c**	报告进程建立活动。
	proc/s	每秒建立的进程数
- **-d**	报告每个块设备的活动。在某些版本的内核下*avgqu-sz, await, svctm, %util*可能不可用，显示为*0.00*。
	tps	每秒到设备的传输数。多个逻辑请求可以合并为一个**物理I/O**请求，每次请求的大小是不定的。
	rd_sec/s	每秒读取的扇区（每扇区512b）数。
	wr_sec/s	每秒写入的扇区数。
	avgrq-sz	平均的请求扇区数。
	avgqu-sz	平均的请求队列数。
	await	I/O请求的平均等待数(ms)。包括队列时间跟处理时间。
	svctm	对I/O请求的平均处理时间。
	%util	处理某个I/O请求花的CPU时间。这个值接进**100%**说明这设备已经饱和了。
- **-f filename**	从**-o filename**创建的文件内获得记录。
- **-I {irq | ALL | SUM | XALL}**	报告某一中断状态。**irq**是中断号。使用多个**-I**选项可以指定多个监控。**SUM**统计每秒中断数。**ALL**报告前16个中断。**XALL**报告所有中断。
- **-n { DEV | EDEV | NFS | NFSD | SOCK | ALL}** 报告网络状态。
 指定**DEV**选项，以下值：
  IFACE	报告的网卡接口名。
  rxpck/s	每秒接收的报文数。
  txpck/s	每秒发送的报文数。
  rxbyt/s	每秒接收的比特数。
  txbyt/s	每秒发送的比特数。
  rxcmp/s	每秒接收的压缩比特数。（cslip等等）
  txcmp/s	每秒发送的压缩比特数。
  rxmcst/s	每秒接收的广播包数。
 指定**EDEV**选项，报告错误状态。
  IFACE 网卡名称。
  rxerr/s	每秒接收报文数。
  txerr/s	每秒发送报文数。
  coll/s	每秒传输过程中发生的碰撞数。
  rxdrop/s	因linux缓存空间不足，每秒丢弃的接收报文数。
  txdrop/s	因linux缓存空间不足，每秒丢弃的发送报文数。
  txcarr/s	每秒的传输错误数。（翻译不准确）
  rxfram/s	每秒接收报文帧错误数。
  rxfifo/s	接收报文每秒的FIFO溢出数。
  txfifo/s	发送报文每秒的FIFO溢出数。
 指定**SOCK**选项，报告**sockets**状态。
  totosck	使用的sockets总数
  tcpsck	tcp使用的sockets数。
  udpsck	udp使用的sockets数。
  rawsck	raw使用的sockets数。
  ip-frag	当前IP分片数
- **-P { cpu | ALL }**	报告CPU状态。**ALL**报告所有CPU状态。
- **-P**	更友好的打印输出。与**-d**一起使用的时候，显示设备全明而不是类似 **dev m-n**这样，参看*/etc/sysconfig/sysstat.ioconf*文件。
- **-q**	报告队列长度和负载。
    run-qz	运行对列（等待允许的进程数）。
    plist-sz	进程表中的进程和线程数。
    ldavg-1	系统一分钟内负载。
    ldavg-5	过去5分钟内负载。
    ldavg-15	过去15分钟内负载。
- **-r**	查看内存和交换分区使用情况
    kbmemfree	空闲内存
    kbmemused	已使用内存
    %memused 	已使用百分比
    kbbuffers	内核用做缓冲区的大小。
    kbcached 	内核用来缓存数据的大小。
    kbswpfree 	空闲交换分区
    kbswpused	交换分区使用
    %swpused	交换分区使用率
    kbswpcad	交换分区已缓存数据。
- **-R**	报告内存状态。
    frmpg/s	内核每秒释放的内核页数。一个**负值**说明内核正在占用内存。个页是4KB还是8KB取决于机器架构。
    bufpg/s	内核每秒用来做缓冲的页数。
    campg/s	内核每秒用来做缓存的页数。
- **-u**	CPU利用率。
    %user	在用户级别（应用）执行使用的CPU。
    %nice	在用户级别（应用）伴随**nice优先级**执行使用的CPU。
    %system	在系统级别（内核）执行使用的CPU。
    %iowait	在等待一个未完成I/O请求的CPU。
    %steal	调度管理程序服务某一虚拟处理器的时候，其他处理器的等待时间
    %idle	系统没有I/O请求的CPU百分比。
- **-v**	报告**i-node、文件、其他内核表**情况。
    dentunusd	目录缓存中没有使用的缓存项。
    file-sz	未使用的文件`handles`
    inode-sz	未使用的i节点`handles`
    super-sz	内核分配的超级块`handles`
    %super-sz	已分配的超级块`handles`占linux最多能分配的百分比
    dquot-sz	已分配的磁盘配额
    rtsig-sz
    %rtsig-sz
- **-w**	报告系统切换活动。
    cswch/s	每秒上下文切换总数
- **-W**	交换分区情况。
    pswpin/s	每秒进入swap的页
    pswpout/s	每秒出去swap的页
- **-x { pid | SELF | ALL }** 报告进程状态。此时**-o、-f无限**，最多报告256个。
    minflt/s	此进程每秒产生的**minor faults**，这不需要从磁盘载入页到内存
    majflt/s	**major faults**，需要从磁盘载入页到内存。
    %user	此程序在用户级别执行使用的CPU，不管有没有**nice priorty**
    %system	系统级别（内核）执行使用的CPU
    nswap/s	内核每秒交换出去的页数。
    CPU	此进程在哪个CPU执行

- **-X { pid | SELF | ALL }** 报告进程的子进程状态。此时**-o、-f无限**，最多报告256个。
    cminflt/s	此进程子进程每秒产生的**minor faults**，这不需要从磁盘载入页到内存
    cmajflt/s	**major faults**，需要从磁盘载入页到内存。
    %cuser	此程序在用户级别执行使用的CPU，不管有没有**nice priorty**
    %csystem	系统级别（内核）执行使用的CPU
    cnswap/s	内核每秒交换出去的页数。
- **-y**	报告tty设备活动
    rcvin/s	当前串行线每秒获得的中断数，串行线号由**TTY列**给出。
    xmtin/s	当前串行线每秒非出的中断数，串行线号由**TTY列**给出。
    framerr/s	当前串行线每秒发生的帧错误。
    prtyerr/s	当前串行线每秒发生的奇偶(parity)错误。
    brk/s	当前串行线每秒发生的**break**
    ovrun/s	每秒溢出
- **-e [hh:mm:ss]**	报告的截止时间（24小时制），只配合**-f**、**-o**选项使用。
- **-s [hh:mm:ss]**	报告的开始时间（24小时制），只配合**-f**、**-o**选项使用。
