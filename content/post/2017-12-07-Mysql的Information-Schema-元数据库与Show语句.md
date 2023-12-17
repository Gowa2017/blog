---
title: Mysql的Information_Schema 元数据库与Show语句
categories:
  - 数据库
date: 2017-12-07 15:16:19
updated: 2017-12-07 15:16:19
tags:
  - MySQL
---
仔细阅读了MySQL的官方参考文档，才会仔细的查看MySQL的这些元数据保存在什么地方。其实全都在**INFORMATION_SCHEMA**数据库内。我们可以从这里看到所有的信息。事实上我们用`show {database | table} ...` 语句展示的就是这里面的内容。
<!--more-->
# 各个表
先看看里面保存的是什么东西。

```sql
mysql> use information_schema;
mysql> show tables;
+---------------------------------------+
| Tables_in_information_schema          |
+---------------------------------------+
| CHARACTER_SETS                        |
| COLLATIONS                            |
| COLLATION_CHARACTER_SET_APPLICABILITY |
| COLUMNS                               |
| COLUMN_PRIVILEGES                     |
| ENGINES                               |
| EVENTS                                |
| FILES                                 |
| GLOBAL_STATUS                         |
| GLOBAL_VARIABLES                      |
| KEY_COLUMN_USAGE                      |
| PARTITIONS                            |
| PLUGINS                               |
| PROCESSLIST                           |
| PROFILING                             |
| REFERENTIAL_CONSTRAINTS               |
| ROUTINES                              |
| SCHEMATA                              |
| SCHEMA_PRIVILEGES                     |
| SESSION_STATUS                        |
| SESSION_VARIABLES                     |
| STATISTICS                            |
| TABLES                                |
| TABLE_CONSTRAINTS                     |
| TABLE_PRIVILEGES                      |
| TRIGGERS                              |
| USER_PRIVILEGES                       |
| VIEWS                                 |
+---------------------------------------+
```
包含了这些内容：**字符集、排序、各字符集使用的排序、列、列权限、引擎、正在执行的事件、打开的文件、全局状态、全局变量、分区、插件、进程信息、PROFILING(不知道什么)、外键约束、过程、数据库信息、数据库权限、会话状态、会话变量、表索引信息、表、表约束、表权限、触发器、用户权限、视图**。
对于某些表，不用什么讲的，你可以用:

```sql
select * from 表名\G;
```
自己观察一下就知道了。
# show 语句
SHOW有多种形式，可以提供有关数据库、表、列或服务器状态的信息。本节叙述以下内容：

	SHOW CHARACTER SET [LIKE 'pattern']
	SHOW COLLATION [LIKE 'pattern']
	SHOW [FULL] COLUMNS FROM tbl_name [FROM db_name] [LIKE 'pattern']
	SHOW CREATE DATABASE db_name
	SHOW CREATE TABLE tbl_name
	SHOW DATABASES [LIKE 'pattern']
	SHOW ENGINE engine_name {LOGS | STATUS }
	SHOW [STORAGE] ENGINES
	SHOW ERRORS [LIMIT [offset,] row_count]
	SHOW GRANTS FOR user
	SHOW INDEX FROM tbl_name [FROM db_name]
	SHOW INNODB STATUS
	SHOW [BDB] LOGS
	SHOW PRIVILEGES
	SHOW [FULL] PROCESSLIST
	SHOW [GLOBAL | SESSION] STATUS [LIKE 'pattern']
	SHOW TABLE STATUS [FROM db_name] [LIKE 'pattern']
	SHOW [OPEN] TABLES [FROM db_name] [LIKE 'pattern']
	SHOW TRIGGERS
	SHOW [GLOBAL | SESSION] VARIABLES [LIKE 'pattern']
	SHOW WARNINGS [LIMIT [offset,] row_count]

还有主从控制的语句：

	SHOW BINLOG EVENTS
	SHOW MASTER LOGS
	SHOW MASTER STATUS
	SHOW SLAVE HOSTS
	SHOW SLAVE STATUS

