---
title: Oracle的 exp/imp
toc: true
date: 2016-11-30 16:23:24
tags: [Oralce, 数据库]
categories: 数据库
---
Oracle数据库的备份，可以通过物化视图，可以通过物理备份（需要关闭数据库），可以通过热备份（归档模式）等方式，但是如果数据量小的时候我可以通过导入/导出数据来进行备份，类似于mysqldump命令。
<!--more-->
# 前言
exp/imp命令具有三种方式（级别）的导出，**全库（FULL）、用户（OWNER）、表（TABLE）**
- 完全（需要具有所有**exp_full_database**权限或者以**特权用户sys/system**）
```sql
EXP SYSTEM/MANAGER BUFFER=34567 FILE=/backup/`date`.dmp DIRECT=y FULL=Y
```
- 用户（用户所有对象被导出）
```sql
	EXP SCOTT/TIGER BUFFER=34567 FILE=/backup/`date`-`id-un`.dmp DIRECT=y OWNER=SCOTT
```
- 表（用户的T1、T2表被导出）
```sql
	EXP SCOTT/TIGER BUFFER=34567 FILE=/backup/`date`-`id-un`.dmp DIRECT=y OWNER=SCOTT TABLES=(T1,T2)
```
> 上面的命令可以用 **imp** 进行替换。 

**direct=y：**直接导出模式，数据直接从磁盘中读取到导出session的UGA中，跳过SQL命令处理层。避免了不必要的数据转换，然后将纪录返回给导出客户端，然后写到导出文件。 
跳过了SQL命令处理层表示DIRECT导出不支持QUERY选项。
设置大的**RECORDLENGTH**值（最大64K）可以减少IO，加快速度。
# exp命令格式

	EXP KEYWORD=value or KEYWORD=(value1,value2,...,valueN)
例子：

	EXP SCOTT/TIGER GRANTS=Y TABLES=(EMP,DEPT,MGR)
	           or TABLES=(T1:P1,T1:P2)  #if T1 is partitioned table
> 我们可以通过 `exp help=y`来看所有的选项

以下选项:**USERID**必须是第一个。

|Keyword    |Description (Default)      |Keyword      |Description (Default)|
|-----------|:-------------------------:|:-----------:|:-------------------:|
|USERID     |username/password          |FULL         |export entire file (N)|
|BUFFER     |size of data buffer        |OWNER        |list of owner usernames|
|FILE       |output files (EXPDAT.DMP)  |TABLES       |list of table names|
|COMPRESS   |import into one extent (Y) |RECORDLENGTH |length of IO record|
|GRANTS     |export grants (Y)          |INCTYPE      |incremental export type|
|INDEXES    |export indexes (Y)         |RECORD       |track incr. export (Y)|
|DIRECT     |direct path (N)            |TRIGGERS     |export triggers (Y)|
|LOG        |log file of screen output  |STATISTICS   |analyze objects (ESTIMATE)|
|ROWS       |export data rows (Y)       |PARFILE      |parameter filename|
|CONSISTENT |cross-table consistency(N) |CONSTRAINTS  |export constraints (Y)|

**更多选项：**
>OBJECT_CONSISTENT    transaction set to read only during object export (N)
>FEEDBACK             display progress every x rows (0)
>FILESIZE             maximum size of each dump file
>FLASHBACK_SCN        SCN used to set session snapshot back to
>FLASHBACK_TIME       time used to get the SCN closest to the specified time
>QUERY                select clause used to export a subset of a table
>RESUMABLE            suspend when a space related error is encountered(N)
>RESUMABLE_NAME       text string used to identify resumable statement
>RESUMABLE_TIMEOUT    wait time for RESUMABLE 
>TTS_FULL_CHECK       perform full or partial dependency check for TTS
>VOLSIZE              number of bytes to write to each tape volume
>TABLESPACES          list of tablespaces to export
>TRANSPORT_TABLESPACE export transportable tablespace metadata (N)
>TEMPLATE             template name which invokes iAS mode export

# imp命令格式

     IMP KEYWORD=value or KEYWORD=(value1,value2,...,valueN)
例子：

     IMP SCOTT/TIGER IGNORE=Y TABLES=(EMP,DEPT) FULL=N
              or TABLES=(T1:P1,T1:P2), #if T1 is partitioned table

以下选项:**USERID**必须是第一个。

|Keyword  |Description (Default)       |Keyword      |Description (Default)|
|---------|:--------------------------:|:-----------:|:-------------------:|
|USERID   |username/password           |FULL         |import entire file (N)|
|BUFFER   |size of data buffer         |FROMUSER     |list of owner usernames|
|FILE     |input files (EXPDAT.DMP)    |TOUSER       |list of usernames|
|SHOW     |just list file contents (N) |TABLES       |list of table names|
|IGNORE   |ignore create errors (N)    |RECORDLENGTH |length of IO record|
|GRANTS   |import grants (Y)           |INCTYPE      |incremental import type|
|INDEXES  |import indexes (Y)          |COMMIT       |commit array insert (N)|
|ROWS     |import data rows (Y)        |PARFILE      |parameter filename|
|LOG      |log file of screen output   |CONSTRAINTS  |import constraints (Y)|

**更多选项：**
>DESTROY                overwrite tablespace data file (N)
>INDEXFILE              write table/index info to specified file
>SKIP_UNUSABLE_INDEXES  skip maintenance of unusable indexes (N)
>FEEDBACK               display progress every x rows(0)
>TOID_NOVALIDATE        skip validation of specified type ids 
>FILESIZE               maximum size of each dump file
>STATISTICS             import precomputed statistics (always)
>RESUMABLE              suspend when a space related error is encountered(N)
>RESUMABLE_NAME         text string used to identify resumable statement
>RESUMABLE_TIMEOUT      wait time for RESUMABLE 
>COMPILE                compile procedures, packages, and functions (Y)
>STREAMS_CONFIGURATION  import streams general metadata (Y)
>STREAMS_INSTANTIATION  import streams instantiation metadata (N)
>VOLSIZE                number of bytes in file on each volume of a file on tape

The following keywords **only apply to transportable tablespaces**
>TRANSPORT_TABLESPACE import transportable tablespace metadata (N)
>TABLESPACES tablespaces to be transported into database
>DATAFILES datafiles to be transported into database
>TTS_OWNERS users that own data in the transportable tablespace set
