---
title: Oracle数据库的当前Redo Log
toc: true
date: 2016-11-30 10:01:38
tags: [Oracle, 数据库]
categories: 
  - 数据库
---
Online Redo Log（常常简称为Redo Log，有别于 Archived Redo Log），是重建数据库（恢复、重新设置）最重要的文件，记录了所有数据库的改变。
一般情况下一个数据库，只有一个`Redo Thread`，但在 __RAC__ 环境下，每个实例具有自己的`Redo Thread`，用以避免对`Redo Log`的竞争，及潜在的性能瓶颈。
<!--more-->
# Redo Log Contents
`Redo Log Files`存储着一系列的`redo records(redo entrys)`。一个`redo record`由一组`change vectors`组成，每个`change vector`记录了数据库中一个`block`的所做的改变。
举例来说，你在一个表`emp`中改变了一个`salary`的值，就会产生一个`redo record`包括描述了`emp`表的`data segment block`，`undo segment block`，和`undo segment`中事务表的变化。
`redo entry`记录了你能用来重建数据库的数据，包括`undo segments`。因此，`redo log`同时也保护`rollback data`。
# 怎么写入Redo Log
`Redo Log`包含两个或多个文件，用来保证始终有一个可写，其他的用来`archived`归档（在数据库是 __ARCHIVELOG__ 模式）。
`LGWR`循环写入`Redo Log Files`，当前使用的文件写满后，就会向下一个可用的文件进行写入，当最后一个写满了，就回到第一开始写。
写满了的文件是否可重用，取决于数据库是否启用`archiving`。
 - `archiving`未启用（ __NOARCHIVELOG__模式），当其中记录的数据变化已经写入`datafile`。
 - `archiving`启用（ __ARCHIVELOG__ 模式），需要其中记录的数据变化已写入`datafile`同时此文件已归档`archived`。
![LGWR Write Redo Log Files](/res/20161130-oracle-redolog-1.png)
## Active(Current)和InActive Redo Log Files
`LGWR`只会将`redo logo buffer`（SGA）中的`redo records`写到一个文件，`LGWR`当前正在写的文件就叫做 __`Current`__ `redo log file`。
实例需要用来进行恢复的就叫做 __ACTIVE__ redo log file，不再需要的就叫做 __INACTIVE__ redo log file。
如果启动了`archiving`（ __ARCHIVELOG__模式），数据库只有重用或者写入一个 __ACTIVE__ 文件直到 __归档进程(ARCn)__已经归档其中内容。
如果没有启动`archiving`（ __NOARCHIVELOG__模式），当最后一个redo log file写满，就会使用第一个 __ACTIVE__的文件进行写入。
## Log Switchs和Log Sequence Numbers
__log switch__ 是指数据库停止写入当前文件，而开始写另外一个。通常是当前文件已经完全写满的时候，当然我们可以手动进行切换。
数据库会分配一个 __log sequence number__ 在发生 __log switch__ `LGWR`开始写入的时候。归档的时候会保留这个 __log sequence number__。
每个 __(online | archive) redo log__ 就是通过 __log sequence number__来识别唯一性。
# Multiplexing Redo Log Files
通过建立`group`，`LGWR`会将同样的数据写到组中的所有成员，这能有效的避免单点故障。
组中的成员要具有相同的大小，放在不同的硬盘上最好，当然放在同一硬盘也是可以的，至少可以避免 __ I/O错误，文件错误__ 等等。
![Multiplexed Redo Log Files](/res/20161130-oracle-redolog-2.png)
当`LGWR`无法写入组中一个成员的时候，数据库会将此成员标注为`INVALID`，并且些一个错误信息到`LGWR`的`trace file`和数据库的`alert log`。
大多数情况下，`group`应该是对称的，就是说具有相同的成员，但你也可以弄来不这样。不过，你必须拥有最少两个以上的`group`否则就是一个错误的配置。
# 创建组
	ALTER DATABASE
	ADD LOGFILE [GROUP 10] ('/oracle/dbs/log1c.rdo', '/oracle/dbs/log2c.rdo') 
	SIZE 500K;
> group number是可选的，当你指定的时候，只能按递增的顺序。
# 添加成员
有时候，组已经建立，只是有的成员可能挂掉了，那么你需要向当前组添加新的成员。

	ALTER DATABASE ADD LOGFILE MEMBER '/oracle/dbs/log2b.rdo' TO GROUP 2;

# 更多参考
参阅[Oracle Administartor's Guide](http://docs.oracle.com/cd/B19306_01/server.102/b14231/onlineredo.htm#i1007497)
