---
title: Oracle使用sqlldr来批量加载数据
categories:
  - 数据库
date: 2020-07-24 12:21:09
updated: 2020-07-24 12:21:09
tags: 
  - 数据库
  - Oracle
---

Mysql 可以直接支持 load data 命令，而 Oracle 似乎没有这样的支持。而且还不支持类似 MySQL 的 `insert into ... values () ` 这样的语句。实在用起来累。

<--more-->

# 拼 SQL 的形式导入数据

# SQLLoader

```sh
用法: SQLLDR keyword=value [,keyword=value,...]

有效的关键字: 

    userid -- ORACLE username/password           
   control -- Control file name                  
       log -- Log file name                      
       bad -- Bad file name                      
      data -- Data file name                     
   discard -- Discard file name                  
discardmax -- Number of discards to allow        (全部默认)
      skip -- Number of logical records to skip  (默认0)
      load -- Number of logical records to load  (全部默认)
    errors -- Number of errors to allow          (默认50)
      rows -- Number of rows in conventional path bind array or between direct path data saves
（默认: 常规路径 64, 所有直接路径）
  bindsize -- Size of conventional path bind array in bytes(默认256000)
    silent -- Suppress messages during run (header,feedback,errors,discards,partitions)
    direct -- use direct path                    (默认FALSE)
   parfile -- parameter file: name of file that contains parameter specifications
  parallel -- do parallel load                   (默认FALSE)
      file -- File to allocate extents from      
skip_unusable_indexes -- disallow/allow unusable indexes or index partitions(默认FALSE)
skip_index_maintenance -- do not maintain indexes, mark affected indexes as unusable(默认FALSE)
commit_discontinued -- commit loaded rows when load is discontinued(默认FALSE)
  readsize -- Size of Read buffer                (默认1048576)
external_table -- use external table for load; NOT_USED, GENERATE_ONLY, EXECUTE(默认NOT_USED)
columnarrayrows -- Number of rows for direct path column array(默认5000)
streamsize -- Size of direct path stream buffer in bytes(默认256000)
multithreading -- use multithreading in direct path  
 resumable -- enable or disable resumable for current session(默认FALSE)
resumable_name -- text string to help identify resumable statement 
resumable_timeout -- wait time (in seconds) for RESUMABLE(默认7200)
date_cache -- size (in entries) of date conversion cache(默认1000)
```

一般我们分别使用控制文件和数据文件比较好，当然也可以在控制文件中指定数据文件。

这样个程序的核心就是控制文件，他控制了我们插入什么数据的行为和方式，已经各种更细致的逻辑。

## 控制文件示例

```
OPTIONS (skip=1,rows=128) 
LOAD DATA
INFILE "loginnames" 
truncate 
INTO TABLE tmp1_rengy 
Fields terminated by "," 
Optionally enclosed by '"' 
trailing nullcols 
(
  id,
  loginname
)
````

# sqluldr2

这里还有一个好用的数据批量导出工具。官方网站打不开了，在 [github.com上找到个地址](https://github.com/wxd237/sqluldr4)

自己编译出来试试。使用的方法如下：

```sh

SQL*UnLoader: Fast Oracle Text Unloader (GZIP, Parallel), Release 4.0.1
(@) Copyright Lou Fangxin (AnySQL.net) 2004 - 2010, all rights reserved.

License: Free for non-commercial useage, else 100 USD per server.

Usage: SQLULDR2 keyword=value [,keyword=value,...]

Valid Keywords:
   user    = username/password@tnsname
   sql     = SQL file name
   query   = select statement
   field   = separator string between fields
   record  = separator string between records
   rows    = print progress for every given rows (default, 1000000) 
   file    = output file name(default: uldrdata.txt)
   log     = log file name, prefix with + to append mode
   fast    = auto tuning the session level parameters(YES)
   text    = output type (MYSQL, CSV, MYSQLINS, ORACLEINS, FORM, SEARCH).
   charset = character set name of the target database.
   ncharset= national character set name of the target database.
   parfile = read command option from parameter file 

  for field and record, you can use '0x' to specify hex character code,
  \r=0x0d \n=0x0a |=0x7c ,=0x2c, \t=0x09, :=0x3a, #=0x23, "=0x22 '=0x27 
```

通过 sql 指定 sql文件，或者通过  query 指定 查询语句，来进行导出。
