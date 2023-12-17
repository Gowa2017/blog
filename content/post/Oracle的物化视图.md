---
title: Oracle的物化视图
toc: true
date: 2016-11-30 11:51:07
tags: [Oracle, 数据库]
categories: 数据库
---
视图是对一系列包含一个或多个表或其他视图的数据的特定描述，一个视图像一个表一样响应一个查询。一个视图可以被视为一个虚拟表，或者一系列的查询，大多数能用表的地方，你也可以视图代替。
视图并不需要存储数据，只需要存储其定义。同时我们不能定义触发器到视图，只能在基表（__base table__）上进行定义。
<!--more-->
# 一个例子
*staff*视图从基表 *employees*而来，但是只包含了基表中的5列。
![An Example of View](/res/20161130-oracle-view-1.png)

# Materialized Views
物化视图是一个可以用总结、结算、复制、分发数据的数据对象，用以存储`一个查询`的结果。这个查询在`FROM`语句后的可以是表、视图、或其他物化视图，我们把他们称做 __主表(master tables，一个复制概念)__或 __detail tables（一个数据仓库概念）__。为了一致，我们权且称之为 __主表(master tables)__，包含主表的数据库就叫 __主库（master databases）__。
> 为了向后兼容，建议使用snapshot，而不是 materialize view

出于复制的目的，物化视图允许你将远程数据库上的数据复制到本地。具有 __高级复制(Advanced Replication)__
# Materialized Views Log
物化视图日志是一个用来关联物化视图与主表的表，用于物化视图的两种刷新方式： __Fast(Incremental)__与 __Synchronous__
其与主表位于同一库的同一结构内，一个表只能用一个物化视图日志。
# Refresh Materialized Views
物化视图有三种更新方式： __快速（增量）、同步、完整刷新__。
- 快速刷新：使用常规的物化视图日志，当__DML__语句更新主表的时候，物化视图日志会保存这些变化，并用来更新基于此表的物化视图。
- 同步刷新：使用一个特殊的物化视图日志__staging log__。__DML__对数据的改变会首先在__staging log__内进行描述，然后再用来更新主表及其物化视图。
- 完整刷新：用在没有物化视图日志的时候，比如你刚开始建立视图需要同步的时候。

快速刷新支持两种物化视图日志：__timestamp-based__和__commit SCN-based__。前一种方式在更新物化视图的时候需要更多的设置，后一种则不需要，因此会提高速度，默认情况下使用的是__timestamp-based__。已建立的视图无法用__alter__进行更改，只drop后重新建立。
同步刷新只支持__timestamp-based__的__staging log__
# 用做复制备份的固化视图
这里我们需要在__LCIMS80BAK__库内备份一些__LCIMS80ZHU__库内的一些表。
## 设置tnsnames.ora

	LCIMS80BAK =
	  (DESCRIPTION =
	    (ADDRESS_LIST =
	      (ADDRESS = (PROTOCOL = TCP)(HOST = 10.168.24.61)(PORT = 1563))
	    )
	    (CONNECT_DATA =
	      (SERVICE_NAME = LCIMS80)
	    )
	  )
	LCIMS80ZHU =
	(DESCRIPTION =
	    (ADDRESS_LIST =
	      (ADDRESS = (PROTOCOL = TCP)(HOST = 10.25.100.17)(PORT = 1521))
	    )
	    (CONNECT_DATA =
	      (SERVICE_NAME = LCIMS80)
	    )
## 创建database link  
```sql
	create database link LCIMS80_GZ_dblink
	connect to absadmin identified by imsadmin32
	USING 'LCIMS80ZHU';
	commit;
```
## 创建物化视图
```sql
	CREATE MATERIALIZED VIEW m_customer
	ON PREBUILT TABLE 
	REFRESH FORCE 
	WITH PRIMARY KEY 
	AS SELECT * 
		FROM m_customer@LCIMS80_GZ_DBLINK;
```
## 创建物化视图日志（LCIMS80ZHU）
```sql
	CREATE MATERIALIZED VIEW LOG ON m_customer WITH  ROWID; 
```
## 进行完整刷新

	sqlplus absadmin/imsadmin61@LCIMS80BAK <<!
		exec dbms_mview.refresh('M_CUSTOMER','C')
		quit
	！
## 定时任务进行增量刷新

	sqlplus absadmin/imsadmin61@LCIMS80BAK <<!
		exec dbms_mview.refresh('M_CUSTOMER$,'F')
		quit
	！
# 补充使用
## 查看物化视图及日志
```sql
	SELECT * FROM DBA_MVIEWS;
	SELECT * FROM DBA_MVIEW_LOGS;
```
