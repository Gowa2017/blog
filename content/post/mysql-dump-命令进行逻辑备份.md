---
title: mysql-dump 命令进行逻辑备份
toc: true
date: 2016-11-21 17:38:02
tags: [MySQL, Linux, 数据库]
categories: 数据库
---
mysqldump经常用来备份数据库成为语句样式。但没有详细的一下具体的办法。
其实很多东西都能在info里面找到非常详细的说明，只是，我们太忙了，只想要一个答案，而无心去看而已，所以呢，就把用得到的都看一下来记录吧。
<!--more-->
# 命令模式

	mysqldump [options] [db_name [tbl_name ...]]
当不指定`tbl_name` 或者使用了 `--databases | --all-databases`， 整个库都会被备份。  
同时，mysqldump __不会备份__ `INFOMATION_SCHEMA` 库，即使你显式的指定。  
## 选项options
### 选项组
__`--opt`__ 同等于 `--add-drop-table --add-locks --create-options --disable-keys --extend-insert --lock-tables --quick --set-charset` 这是一个默认的选项组。
__`--compact`__ 同等于 `--skip-add-drop-table --skip-add-locks --skip-comments --skip-disable-keys --skip-set-charset`
### 选项的反转
我们可以用`--skip-opt or --skip-compact` 来进行反转以上选项。
当我们想对--opt组中的某些选项进行__取消__的时候，我们可以用`--skip`来操作:
比如，要取消`extend inserts` 跟 `memory buffing`:

	[--opt] --skip-extend-inserts --skip-quick。
>（--opt可选是因为其是默认开启的）。

如果我们要__取消__`--opt`中的除禁止索引跟锁表外的选项，则可以用:

	--skip-opt --disable-keys --lock-tables。
如果是在组选项中要选择性的关闭或者开启一些功能，顺序很重要。

	--disable-keys --lock-tables --skip-opt 
将不会获得你想要的效果。

### 可能的缓存问题
mysqldump 可以逐行的获取-备份表内容（至文件），同时可以在备份（至文件）前将整个内容缓存到内存中。
数据过大的时候开启缓存会产生问题，想要逐行的进行备份，用`--quick`(或--opt，组选项中开启了--quick)，想要开启缓存那么用`--skip-quick`即可

### 更多的选项
`--help, -?` 显示帮助信息
`--add-drop-database` 在`CREATE DATABASE..`语句前加上`DROP DATABASE ...`。典型的使用是跟`--all-databases` 或`--databases`。
`--add-drop-table` 在`CREATE TABLE ...`语句前加上`DROP TABLE ...`。
`--add-locks` 在每个导出语句前后加上`LOCK TABLES`与`UNLOCK TABLES`，这样插入速度会更快。 更多见 __7.3.2节 INSERT语句的速度__
`--all-databases,-A` 与`--databases 所有库名... `等价，备份所有库，表[infomation_schema除外]
`--allow-keywords` 允许建立field名为关键字的列，在每列加上表名作为前缀。
`--character-set-dir=path` 字符集安装路径，更多见 __9.5 字符节设置__
`--comments,-i` 默认开启，在备份文件写入版本、主机等信息。`--skip-comments` 关闭
`--compact` 更紧凑的输出。将会开启`--skip-add-drop-table --skip-add-locks --skip-comments --skip-disable-keys --skip-set-charset`
`--compatible=name` 为了与其他数据库系统兼容，`name={ansi | mysql323 | mysql40 | postgresql | oracle | msssql | db2 | maxdb | no_key_options | no_table_options | no_field_options}`多个选项的话用逗号分开。但这并不能保证就一定能与其他数据库兼容，只是保证输出尽量兼容。更多见 __5.1.6 服务器SQL模式__
`--complate-insert,-c` 使用包含列的完整插入语句。
`--compress,-C` 如果clien and server 支持压缩的话就压缩所有信息。
`--create-options` 包含所有MySQL `CREATE TABLE` 语句支持的表选项
`--databases,-B` 多个库
`--debug[=debug_options],-#[debug_options]` 调试日志，`debug_options`典型格式`d:t:o,file_name`,默认值`d:t:o,/tmp/mysqldump.trace`
`--debug_info` 在程序退出的时候打印 __内存和CPU的使用情况__。 5.0.32增加
`--cdefault-character-set=charset-name` 用charset-name 作为默认的字符集。如果没指定charset-name的话，默认使用utf8,早些版本用的是latin1。此选项对用`--tab`选项产生的文件没有影响。
`--delayed-insert` 用`INSERT DELAYED` 代替 `INSERT`
`--delete-master-logs` 在主从服务器上，在备份操作完成后会给服务器一个`PURGE BINARY LOGS`语句，删除二进制日志。此选项自动开启`--master-data`
`--disable-keys, -K`  用`/*!40000 ALTER TABLE tbl_name DISABLE KEYS */;`和`/*!40000 ALTER TABLE tbl_name ENABLE KEYS*/`包围`INSERT`。这会让插入的时候更快，因为所有索引都在所有行插入完毕后建立。只有在非唯一索引的MyISAM表有效。
`--dump-date` 在`--comments` 开启的情况下，默认在文件尾加上一条注释`-- Dump complete on DATE`。`--skip-dump-date`关闭，默认情况开启。
`--extend-insert, -e` 使用多行的`INSERT`语句包含不止一条的值列表。会产生更小的`dump file`和更快的插入速度。
`--fields-terminated-by=...,--fields-enclosed-by=...,--fields-optionally-enclosed-by=...,--fields=escaped-by=...` 与`--tab`选项同用，与`LOAD DATA INFILE`有相同意义。
`--first-slave` 5.5已删除，使用`--lock-all-tables`替代。
`--flush-logs, -F` 备份前刷新日志，需要`RELOAD`权限。当与`--all-databases`，每个库被备份时都会刷新。当使用`--lock-all-tables or --master-data`，日志只有在所有表在锁定的时候被刷新一次。想要备份跟日志一起刷新，就要配合`--lock-all-tables or --master-data`。
`--flush-privileges` 备份库后发送`FLUSH PRIVILEGES`语句到服务器。当其他有库依赖于此库进行恢复的时候必须使用。
`--force, -f` 出错依然继续。
`--host=host_name, -h host_name` 指定主机，默认`localhost`
`--hex-blob` 二进制列用16进制表示(`abc`->`ox616263`)，受影响的数据类型是`BINARY,VARBINARY,BLOB`
`--ignore-table=db_name.tbl_name` 忽略表或视图，多张表就多次使用。
`--insert-ignore` `INSERT IGNORE ...` 代替`INSERT`语句
`--lines-terminated-by=...`  与--tab 选项联用
`--lock-all-tables, -x` 所有库的表锁定。通过在备份期间产生一个`read lock`全局读锁，会自动关闭`--single-transaction`和`--lock-tables`。
`--lock-tables, -l` 每个库备份前锁表。MyISAM表用`READ LOCAL`锁定来允许同步插入。事务性的表（InnoDb/BDB)，`--single-transaction`选项更好，因为不需要锁表。 因为`--lock-tables`为每个库单独锁表，并不能保证在备份在文件后的内容在库间逻辑上的一致。
`--log-error=file_name` 默认情况下不作记录，此选项将警告或错误附加到`file_name`
`--single-transaction` 备份前发送`START TRANSACTION SQL`语句给服务器，对事务性的表很有用。当`BEGIN`被执行的时候，其备份各库的一致状态，且不会阻塞任何应用。必须记住的是，只有事务表（`innoDB/BDB`）才会在一致状态下备份。假入是MyISAM，其状态依然会有可能改变。在备份期间，为了保证成功（备份文件正常、二进制文件坐标正确），其他连接不可使用任何一个： `{ ALTER TABLE | CREATE TABLE | DROP TABLE | RENAME TABLE | TRUNCATE TABLE}`语句。因为一致性读与这些语句是不隔离的， 在要备份的表上执行这几个语句会使 mysqldump执行的 `SELECT`命令重新读取表内容而得到错误的内容或者失败。`--single-transaction`与`--lock-tables`是互斥的，因为`--lock-tables`会让所有未提交的事务悄悄的提交。对于较大的表，应该结合`--quick`使用。
`--master-data[=value]` 产生的备份文件可以作用当前主从服务器的从属服务器。这将导致备份出来的文件含有一个`CHANGE MASTER TO`语句,指明二进制日志 （file name 和 position)与被备份服务器一致。 在导入从服务器后可以从主服务器日志处开始同步复制。 默认是value是1,如果值是2,只是写在信息式的写在注释内，并不生效。 此选项要求`RELOAD`权限和二进制日志必须开启。`--master-data`自动关闭`--lock-tables`。在`--single-transaction`没指定的情况下开启`--lock-all-tables`（这样情况下全局读锁只会在开始备份的产生很短的时间）。无论什么情况，备份的时刻日志就会产生动作。
在已有的从服务器上备份出文件再做一个从服务器也是可以的。

1. 停止从服务器进程，获取其状态：
```
	mysql > STOP SLAVE SQL_THREAD;
	mysql > SHOW SLAVE STATUS;
```
2. 从上面得到主服务器二进制日制的坐标：`Relay_Master_Log_File Exec_Master_Log_Pos`字段，把他给记做`file_name file_pos`。
3. dump 从服务器。
```
	shell > mysqldump --master-data=2 --all-database > dumpfile	  
```
4. 重启从服务器线程
```
	mysql > START SLAVE;	   
```
5. 新从服务器，导入dumpfile.
```
	shell > mysql < dumpfile	      
```
6. 在新从服务器上，设置同步坐标。(file_name file_pos)
```
	mysql> CHANGE MASTER TO MASTER_LOG_FILE=file_name MASTER_LOG_POS=file_pos;	   
```

自行添加CHAGE MASTER TO 需要的参数，比如主服务器地址等。
`--no-autocommit` 以`set autocommit=0` 和`COMMIT`语句来包围`INSERT`语句。
`--no-create-db, -n` 当`--databases,-B | --all-databases`指定的时候，禁用`CREATE DATABASES`语句
`--no-create-info, -t` 不写出重新创建备份表的`CREATE TABLE`语句
`--no-data, -d` 只要表结构。
`--no-set-names, -N` 等价于`--skip-set-charset`
`--order-by-primary` 当主键、或唯一索引存在的时候以此排序列行。当MyISAM表要转到InnoDB表的时候，但会花费更多时间。
`--password[=password],-p[password]` 密码
`-pipe, -W`
`--port=port_num, -P port_num`
`--quick, -q` 对于dump大表很实用。强制mysqldump一次获取一行而不是获得整个记录并且进行缓存。
`--quote-names, -Q` 
`--result-file=file_name, -r file_name` 重定向至文件
`--routines, -R` 包括函数跟结构。但没有时间戳，新导入的时间就是重新建立过程的时间戳。想要原来的时间戳，导出mysql.proc表后进行恢复。
`--set-charset` 将`SET NAMES default_character_set`加到输出中。该选项默认启用。要想禁用SET NAMES语句，使用`--skip-set-charset`。
`--skip-comments` 参见`--comments`选项的描述。
`--socket=path, -S path` 当连接localhost(为默认主机)时使用的套接字文件。
`--tab=path, -T path` 产生tab分割的数据文件。对于每个转储的表，mysqldump创建一个包含创建表的`CREATE TABLE`语句的`tbl_name.sql`文件，和一个包含其数据的`tbl_name.txt`文件。选项值为写入文件的目录。
    默认情况，.txt数据文件的格式是在列值和每行后面的新行之间使用tab字符。可以使用`--fields-xxx`和`--line--xxx`选项明显指定格式。
    注释：该选项只适用于mysqldump与mysqld服务器在同一台机器上运行时。你必须具有FILE权限，并且服务器必须有在你指定的目录中有写文件的许可。
`--tables` 覆盖---database或-B选项。选项后面的所有参量被看作表名。
--tariggers --skip-triggers
`--user=user_name, -u user_name` 用户名
`--verbose, -v` 冗长模式。打印出程序操作的详细信息。
`--version, -V` 显示版本信息并退出。
`--xml, -X` 转存为xml文件
`--where` 进行条件过滤。比如我想要将某张表内日期大于某个时间的数据进行导出：`mysqldump -u -p  database table --where "create_date > '2020-02-19' and update_date > '2020-02-19'" >> out.sql`。注意值要有 `''` 包起来。
