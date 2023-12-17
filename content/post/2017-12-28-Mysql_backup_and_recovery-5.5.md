---
title: Mysql-Backup-and-Recovery 中文翻译
categories:
  - 数据库
date: 2017-12-28 22:16:19
updated: 2017-12-28 22:16:19
tags:
  - MySQL
---
本文来源于对Mysql 5.5官方手册的翻译版本，逐渐更新中。
<!--more-->
# 前言
备份对于在出现系统崩溃、 硬件故障、错误删除数据等情况下进行数据恢复是非常重要的。同时，备份也可以作为升级的时候做一个数据保障，或者用来设置一个复制服务器。

MySQL提供多种备份方式，你可以根据自己的需求来进行选择怎么备份。本章谈论几个你可能已经熟悉了的关于备份与恢复主题：
  - 备份方式： 逻辑 VS 物理，完整 VS 增量等等。
  - 创建备份的方式。
  - 恢复的方法，包括时间点恢复
  - 备份调度，加密，压缩。
  - 表维护，对于异常表的恢复。

# 备份和恢复的方式

# 物理备份（RAW）与逻辑备份
物理备份由组成数据库的目录和文件的原始副本构成。这种方式适合大型数据库和需要快速恢复的重要数据库的备份。

逻辑备份以逻辑数据库信息存储数据，也就是一系列的SQL语句（`CREATE DATABASE, CREATE TABLE`）和内容（`INSERT`）。这种方式适合数据量不大，或者你想要进行修改数据和在其他机器上进行重新建库的业务场景。

**物理备份有以下特点：**

 - 包括数据文件和目录的备份。典型地，可以是数据目录的全部或者部分。
 - 比逻辑备份快，因为只进行文件复制而不用进行转换。
 - 输出更紧凑。
 - 对于繁忙、重要的数据库，速度和压缩是非常重要的，MySQL Enterprise Backup Product采用的就是物理备份。
 - 备份和恢复的粒度从整个数据目录到单个文件。根据存储引擎的不同，有可能会提供表级别的粒度。举例说，`Innodb`每个表可以在一个数据文件内，也可以共享同一个表空间；`MyISAM`的话，每个表会使用好几个文件。
 - 作为补充，这个备份可以包含所有的日志文件或者配置文件。
 - 使用`MEMORY`引擎的表是很难备份的，因为他们并不存储在磁盘上。（MySQL Enterprise Backup Product有个选项可以让你备份的时候从`MEMORY`引擎的表内获取数据。）
 - 备份只能在具有相同的架构或特点的机器上是可移植的。
 - 物理备份工具包括 Mysql Enterprise Backup 的**mysqlbackup**（可以备份`InnoDB`表和其他任何类型表），文件系统级别的命令（`cp，scp，tar, rsync`）,还有对`MyISAM`表使用的**mysqlhotcopy**工具。
 - 恢复：
   MySQL Enterprise Backup可以恢复`InnoDB`表和任何其他其备份的表。
   `ndb_restore`恢复`NDB`表。
   在文件系统级别进行复制的备份或者用**mysqlhotcopy**的备份可以用文件系统命令将其拷贝回原来的地方。

**逻辑备份特点：**  

 - 通过查询服务器来获得数据库的结构和内容信息。
 - 备份的速度比物理备份慢。因为服务器必须访问数据库信息并且转换为逻辑格式。如果输出是在客户端，那么服务器还需要把这些数据发送给客户端。
 - 备份输出比物理备份大，因为是以`text`格式进行存储。
 - 备份和恢复的粒度从系统级（所有库）、 数据库级（某一库的所有表）、到表级别。这个是无论表使用的是什么引擎。
 - 以逻辑备份格式存储的备份是与机器无关、高度可移植的。
 - 逻辑备份可以在服务器运行时进行，不用关机下线。
 - 逻辑备份工具包括 `mysqldump` 和`SELECT ... INTO  OUTFILES`语句。这可以对所有引擎使用，包括`MEMORY`引擎表。
 - 为了恢复逻辑备份，SQL格式的转储文件可以使用**mysql**客户端进行，对于text格式的转储文件，可以使用 `LOAD DATA INFILE`或者 **mysqlimport**工具。

## 在线 VS 离线备份
本地备份是在服务器运行的机器上进行备份，远程备份的话就是在非服务器运行的另外一台机器上进行备份。对于某些类型的备份，可以在远程机器上执行，然后输出到服务器主机上。

 - **mysqldump**可以连接到本地或者远程主机。对于 SQL语句的输出（`CREATE`和`INSERT`语句），本地或者远程的都能成功，并且在客户端产生输出。对于格式化的文本输出（`--tab`选项），数据文件只在服务器上产生。
 - **mysqlhotcopy**只进行本地备份：其连接到服务器并进行锁定以避免数据的修改，然后复制表数据文件。
 - `SELECT ... INTO OUTFILE`可以在本地或者远程执行，但是其输出是在服务器上。
 - 物理备份一般是在本地执行的，这样服务器就可以下线，及时我们备份后是要输出到其他服务器。

## 快照备份
某些系统实现启用了“快照”功能。这允许在某一时间对文件系统进行逻辑复制而不用对整个文件系统进行物理复制。（比如，系统实现可能会使用 **copy-on-write**技术只复制在快照备份后产生的变化）。MySQL本身并不提供文件系统快照。其是通过第三方解决方案达成的，如 Veritas, LVM, ZFS。    

**完整 VS 增量备份**

一个完整备份包括在某一时间点由MySQL服务器所管理的所有文件。增量备份包括在某一时间间隔内的数据变化。MySQL包括不同的完整备份方式，如前文所述。增量备份通过启用服务器的二进制日志（服务器用来记录数据改变的）。

**完整 VS 时间点（增量）恢复**

一个完整恢复从一个完整备份还原数据，这会将数据库实例还原到其备份的状态。如果完整备份不同达到想要的状态，那么接下来会进行一个增量备份的恢复，以保证实例到达最新状态。

增量恢复就对一个在时间间隔内的备份进行恢复。这也被叫做时间点恢复，因为它将使服务器从当前状态转变到一个时间点时的状态。时间点恢复是以二进制日志为基础的，典型情况下其都是跟随在一个完整备份之后，这样写在二进制日志里的数据变化就会被重放，以让数据库到达希望的时间点状态。

## 表维护
在表出现故障的时候数据完整性无法得到保障。对于 `InnoDB`来说，这不是一个问题。对于在`MyISAM`出现问题是进行检查和修复的工具，查看**7.6节 MyISAM表维护及崩溃恢复。**  

**备份调度，压缩，加密**
备份调度对于自动备份来说是非常有用的。压缩的会使备份占用更少的磁盘空间，加密能提供更好的安全性以避免未授权的对备份数据的访问。MySQL本身不提供这些功能。MySQL Enterprise Backup可以压缩`InnoDB`备份，能通过系统工具集进行加密或者压缩。有某些第三方工具可以进行使用达成这样的目的。

# 数据库备份方式
这一节介绍了一些常用的备份方式。

**用MySQL Enterprise Backup 进行热备份**

Customers of MySQL Enterprise Edition can use the MySQL Enterprise Backup product to do physical backups of entire instances or selected databases, tables, or both. This product includes features for incremental and compressed backups. Backing up the physical database files makes restore much faster than logical techniques such as the mysqldump command. InnoDB tables are copied using a hot backup mechanism. (Ideally, the InnoDB tables should represent a substantial majority of the data.) Tables from other storage engines are copied using a warm backup mechanism. For an overview of the MySQL Enterprise Backup product, see Section 25.2, “MySQL Enterprise Backup Overview”.

**用mysqldump 或 mysqlhotcopy备份**

**mysqldump**可以备份任意类型的表，所以更为常用，而**mysqlhotcopy**只能备份某些存储引擎表。对于`InnoDB`表，通过`--single-transaction`选项可以实现在线无锁表的备份。

**通过复制文件实现备份表**

对于某些存储引擎类型的表，每个表使用其自己的数据文件，就可以使用复制文件的方式进行备份。**MyISAM**表就是以文件的形式进行存储的，所以很方便的就可以通过复制文件进行备份（*.frm, *.MYD, *.MYI文件）。为了获得一致性的备份，**停止服务器**或者**锁定办刷新要备份的表**：

 	FLUSH TABLES tbl_list WITH READ LOCK;
 	
只需要读锁；这样就允许其他客户端在你进行备份的时候继续查询数据库。**刷新**是必须的，这用来保证在备份之前所有活跃的索引页都被写入磁盘。

我们也可以在服务器没有进行任何更新的时候复制所有的表文件来制造一个二进制备份。**mysqlhotcopy**就是这样做的。

> 注意：在包含**InnoDB**表的时候，这样做是没有意义的。**mysqlhotcopy**无法在`InnoDB`表上工作，因为并不一定把数据存储在数据目录内。而且，尽管服务器没有更新数据，`InnoDB`依然可能有改变的数据在内存而还未写至磁盘。

**格式化的文本文件备份**

如果要获得一个包含表数据的文本文件，可以使用 `SELECT * INTO OUTFILE 'file_name' FROM tbl_name`。这个输出会产生在服务器上，而不是在客户端。输出的文件必须是不存在的，因为允许重写已存在的文件是一个安全隐患。这种方式对于任何形式的表都能工作，但不存储表结构。

另外一个用来获得文本数据格式（包括`CREATE TABLE`语句）的方法是使用**mysqldump**加上`--tab`选项。

为了重新载入一个文本格式的备份，使用`LOAD DATA INFILE`或者**mysqlimport**。

**通过启用二进制日志获得增量备份**

MySQL支持增量备份：必须为服务器开启`--log-bin`选项来启用二进制日志。二进制日志记录了所有的数据变化信息。在需要进行增量备份的时刻（包含所有自上次完整或增量备份以来的变化），我们可以用`FLUSH LOGS`来回转日志文件。这样，就可以把想要备份的文件复制到备份位置。这些二进制日志就是增量备份。至于在恢复的时候怎么使用，参考**7.5 通过二进制日志进行时间点恢复**。在下一次进行完整备份的时候，我们一样要回转日志，用`FLUSH LOGS`, `mysqldump --flush-logs`, `mysqlhotcopy --flushlog`。参考**4.5.4 mysqldump--一个数据库备份程序**， **4.6.9 mysqlhotcopy-- 一个数据库备份程序**。

**通过复制进行备份**

如果在进行主服务器备份的时候产生了性能问题，一个好的建议就是设置从服务器并且从服务器上进行备份操作。**17.3.1 用复制来进行备份**。

在备份从服务器的时候，不管采用什么方式，也要备份`master.info`和`relay-log.info`文件。这些信息在你重新设置从服务器的时候总是需要的。如果在备份的时候，从服务器正在执行`LOAD DATA INFILE`语句，那就要备份所有的从服务器载入的文件，因为备份后从服务器需要用这些文件来重新进行载入以继续`LOAD DATA INFILE`操作。一般来说，这个文件的位置是在`tmpdir`系统变量中，或者由`--slave-load-tmpdir`选项在服务器启动时指定。

**恢复故障表**

如果要恢复出现故障的**MyISAM**表，尝试使用`REPARI TABLE`或者`myisamchk -r`，这在99.9%的情况下都能工作。如果**myisamchk**失败了，参考**7.6节 MyISAM表维护和崩溃恢复**。

**用文件系统快照进行备份**

如果在使用**Veritas**文件系统，可以按如下步骤做：  
1. 从一个客户端，执行 ` FLUSH TABLES WITH READ LOCK;`；  
2. 从另外一个**shell**，执行 ` mount vxfs snapshot`；  
3. 从 **第1.**部的客户端，执行`UNLOCK TABLES；`；  
4. 从快照复制文件。  
5. 卸载快照。

类似的快照能力在其他文件系统可能也有，比如**LVM, ZFS**。

# 备份和恢复策略举例
这一节讨论了如何备份，以便在遇到一些类型的错误后进行恢复。

 - 操作系统崩溃
 - 电源故障
 - 文件系统崩溃
 - 硬件问题（驱动，主板等等）

示例的命令中，`mysqldump`和`mysql`不包含`--user`或`--password`选项。当你自己在使用的时候，你需要加上必要的选项来连接上服务器。

假设，数据存储在`InnoDB`引擎，它支持事务和自动崩溃恢复。

对于操作系统故障或者电源故障的情况，我们可以假设MySQL的磁盘数据在重启后是可用的。因为崩溃，`InnoDB`数据文件不一定包含一致的数据，但`InnoDB`会阅读其重做日志，并且找出那些没有并刷新到数据文件的挂起的提交事务和未提交的事务。`InnoDB`会自动回滚未提交的事务，刷新提交了的事务到数据文件。下面是一个可能的例子：

```
InnoDB: Database was not shut down normally.
InnoDB: Starting recovery from log files...
InnoDB: Starting log scan based on checkpoint at
InnoDB: log sequence number 0 13674004
InnoDB: Doing recovery: scanned up to log sequence number 0 13739520
InnoDB: Doing recovery: scanned up to log sequence number 0 13805056
InnoDB: Doing recovery: scanned up to log sequence number 0 13870592
InnoDB: Doing recovery: scanned up to log sequence number 0 13936128
...
InnoDB: Doing recovery: scanned up to log sequence number 0 20555264
InnoDB: Doing recovery: scanned up to log sequence number 0 20620800
InnoDB: Doing recovery: scanned up to log sequence number 0 20664692
InnoDB: 1 uncommitted transaction(s) which must be rolled back
InnoDB: Starting rollback of uncommitted transactions
InnoDB: Rolling back trx no 16745
InnoDB: Rolling back of trx no 16745 completed
InnoDB: Rollback of uncommitted transactions completed
InnoDB: Starting an apply batch of log records to the database...
InnoDB: Apply batch completed
InnoDB: Started
mysqld: ready for connections
```

对于文件系统崩溃或者硬件问题，我们假设在服务器重启后，磁盘数据是不可用的。这就意味着MySQL会启动失败，因为部分磁盘块已经不可读。这样的情况下，只能重新格式化磁盘，或者解决硬件问题等。然后通过备份来进行恢复，这样就要求数据已经备份。那么，为了保证在这样的情况下不会丢失数据，设计和实现一个备份策略。

### 和备份策略通信

为了实用，备份必须是常规性的。一个完整备份（某一时间点的数据快照）可以用几个工具来做到。例如，MySQL Enterprise Backup可以对一个实例进行物理备份，然后进行优化以避免在备份`InnoDB`的时候出现中断；**mysqldump**提供在线的逻辑备份。这里我们用`mysqldump`进行讨论。

假设我们要做一个对所有数据库的所有`InnoDB`表在星期1的下午1点进行一个完整备份：

	shell> mysqldump --single-transaction --all-databases > backup_sunday_1_pm.sql

生成的 *.sql 文件包含了一系列的`INSERT`语句可以在后面的时间内用来插入数据。

这个备份操作在开始之前需要一个全局的对所有表的读锁。（`FLUSH TABLES WITH READ LOCK`）。当这个锁获得后，这个二进制日志的坐标就会被记录，然后释放锁。如果在`FLUSH`语句执行期间正在执行一个耗时很长的更新语句，那么这个操作就只有到更新完毕才会进行。在此之后，这个过程就不再需要锁，同时也不会打扰其他客户端的读与写了。

先前假设我们讨论的表是`InnoDB`表，所以`--single-transaction`使用一致性读并且保证被`mysqldump`看到的数据不会改变（其他客户端所做的变化mysqldump看不到）。如果备份中包括非事务表，那么必须保证在这过程中表不会改变。比如，在`mysql`库内的所有MyISAM表，必须没有对MySQL账号管理的改变。

完整备份是必要的，但是很多时候其并不方便。它会产生很大的备份文件并且耗时不少。每一个完整备份都包含备份之间没有改变的数据，这是非常低效的。更高效的做法是先做一个初始的完整备份，再产生增量备份。增量备份更小，更快。但是相对的，在恢复的时候就不能只恢复完整备份，同样需要恢复增量备份。

为了制作增量备份，我们就要备份增量变化。MySQL是以二进制日志来进行实现的，所以`mysqld`应该总是以`--log-bin`选项启动。这样在更新数据的时候就会把数据变化写到二进制日志内。查看一下已经以`--log-bin`选项运行了几天的MySQL的数据目录，可以找到几个二进制日志：

```
-rw-rw---- 1 guilhem  guilhem   1277324 Nov 10 23:59 gbichot2-bin.000001
-rw-rw---- 1 guilhem  guilhem         4 Nov 10 23:59 gbichot2-bin.000002
-rw-rw---- 1 guilhem  guilhem        79 Nov 11 11:06 gbichot2-bin.000003
-rw-rw---- 1 guilhem  guilhem       508 Nov 11 11:08 gbichot2-bin.000004
-rw-rw---- 1 guilhem  guilhem 220047446 Nov 12 16:47 gbichot2-bin.000005
-rw-rw---- 1 guilhem  guilhem    998412 Nov 14 10:08 gbichot2-bin.000006
-rw-rw---- 1 guilhem  guilhem       361 Nov 14 10:07 gbichot2-bin.index
```
每当`mysqld`重启的时候就会创建一个新的二进制文件，序号会递增。在服务器运行中你也可以让服务器关闭当前的二进制日志而重新建立一个，使用`FLUSH LOGS`语句或`mysqladmin flush-logs`。`mysqldump`也有类似的选项。 **.index**文件记录了所有的二进制文件名列表。

二进制文件组成了增量备份，所以是非常重要的。如果你确定要在完整备份的时候刷新日志，后面建立的二进制日志就包含了所有你完整备份后的数据变化信息。让我们来修改一下先前的代码，这次我们会刷新日志，这样转储后的文件就会包含下一个二进制的文件名：

	shell> mysqldump --single-transaction --flush-logs --master-date=2\
	       --all-databases > backup_sunday_1_pm.sql

在执行这个命令之后，数据目录下包含了一个新的二进制日志`gbichot2-bin.000007`，因为`--flush-logs`选项让服务器进行了刷新。`--master-data`选项让`mysqldump`将二进制日志信息写到输出，所以这次生成的 .sql 文件包含：

	-- Position to start replication or point-in-time recovery from
	-- CHANGE MASTER TO MASTER_LOG_FILE='gbichot2-bin.000007',MASTER_LOG_POS=4;

因为`mysqldump`做了一个完整备份，这两行意味着：
 - 转储文件包含在`gbichot2-bin.000007`前的所有数据及变化。
 - 所有在备份后产生的数据变化都写在`gbichot2-bin.000007`日志内。

在周一下午1点，我们可以通过刷新日志产生一个新日志来进行增量备份。`mysqladin flush-logs`命令将会产生`gbichot28-bin.000008`。所有从周日下午一点到周一下午一点的数据变化都在`gbichot2-bin.000007`。增量备份是重要的，所以要把他复制到一个安全的地方。
周二下午一点，可以同样执行`mysqladmin flush-logs`来产生一个新的日志`gbichot2-bin.000009`，这样从周一下午一点到周二下午一点的所有数据变化都在`gbichot2-bin.000008`内了。

二进制日志非常占磁盘空间。为了释放空间的话，就要及时删除不用的二进制日志文件，在做了一个完整备份后：

	shell> mysqldump --single-transactio --flush-logs --master-data=2 \
	--all-databases --delete-master-logs > backup_sunday_1_PM.sql

> 注意：用`--delete-master-logs`选项在主服务器上是非常危险的，因为无法保证从服务器已经把所需要的二进制文件读取完毕。`PURGE BINARY LOG`语句解释了在删除二进制日志之前需要确认的事项。**13.4.1.1 PURGE BINARY LOGS 语法**

### 用备份进行恢复

现在，假设我们在周三早上8点发生了一个意外，此时需要进行用备份进行恢复。首先我们进行完整备份的恢复：

	shell> mysql < backup_sunday_1_pm.sql

这时，数据已经存储成为周一下午1点的状态。要恢复从这个时候开始的数据变化，必须使用增量备份；也就是`gbichot2-bin.000007`和`gbichot2-bin.000008`文件。

	shell> mysqlbinlog gbichot2-bin.000007 gbichot2-bin.000008 | mysql

这样我们就已经把数据恢复到了周二的下午1点，但还缺少从这个时间点到崩溃点的数据。为了不丢掉这些数据，我们必须已经让mysql把二进制日志存储到了一个安全的地方（RAID，SAN ...）反正不能和出现问题的硬盘放在一起。如果这样做了。我们现在应该是有一个`gbichot2-bin.000009`或者更大序号的文件，我们可以继续这样：

	shell> mysqlbinlog gbichot2-bin.000009  | mysql

更多使用`mysqlbinlog`的信息，请参考**7.5 用二进制进行时间点（增量）恢复**。

## 备份策略总结

在操作系统或者电源故障，`InnoDB`会自己进行恢复工作。但是为了保证你自己睡得安逸，确认一下下面的事项：

 - 总是让`mysqld`以 `--log-bin`选项或者`--log-bin= _log_name_`运行，并且二进制要指定在与数据目录不一样的安全的磁盘上。如果这样做了，这也是一个非常不错的磁盘负载均衡。
 - 进行间段性的完整备份。用`mysqldump`来进行一个在线的，不阻塞的备份。
 - 进行间段性的增量备份，同时`FLUSH LOGS`或者`mysqladmin flush-logs`。

## 用mysqldump进行备份
这一节描述了怎么样用`mysqldump`来产生转储文件，已经怎么样使用它。有下面几种方式来使用转储文件：

 - 数据恢复
 - 复制设置
 - 实验

`mysqldump`产生两种形式的输出，取决于是否指定了`--tab`选项。

 - 没有`--tab`选项。mysqldump把SQL语句写到标准输出这些输出由`CREATE`语句来创建转储对象（数据库，表，过程等等），`INSERT`语句来载入数据到表。这个输出可以存储到一个文件然后被用`mysql`用来重建数据对象。
 - 有`--tab`。将会为每个dump产生两个文件。一个个 .sql文件，包含`CREATE TABLE`语句；一个 .txt 文件，表的每个记录一行。

### Dump SQL格式数据

	shell> mysqldump [ arguments ] > file_name
备份所有库：

	shell> mysqldump --all-databases > dump.sql
备份指定库：
	
	shell> mysqldump --databases db1 db2 db3 > dump.sql

在没有 --databases 的情况下，db1 被认为是库，而db2 db3会被看做是表。

在使用`--databases`或`--all-databases`的情况下，mysqldump会将 `CREATE DATABASE`和`USE`语句写在每个使用库前。这样就保证了在重新载入的时候，没有库就建立库，然后让这数据从哪里来的，就到哪个库去。如果想要在重建之前强制删除掉库，加上`--add-drop-databas`选项。

备份某一个库：

	shell> mysqldump --databases test > test.sql
	shell> mysqldump  test > test.sql

这两个语句的不同是：后者将不会产生`CREATE DATABASE`语句和`USE`语句。那么：

 - 在使用dump文件的时候，必须指定库名。
 - 可以指定与原来库名不一样的库。
 - 如果库不存在，你必须先手动建立。
 - 因为不产生`CREATE DATABASE`语句，所以`--add-drop-database`是无效的。

备份某库的某些表，在库名后指定表名即可：

     	shell> mysqldump test t1 t3 t7 > dump.sql

### 重载SQL格式的备份
为了重载一个转存文件，将它作为`mysql`客户端的输入即可。如果，在生成转储文件的时候，指定了`--all-databases`或`--databases`选项，文件内会包含`CREATE DATABASE`和`USE`语句，这样就不用在使用的时候指定库名了。

	shell> mysql < dump.sql

或者可以在登录mysql后：

	mysql> source dump.sql

如果要指定库名：

	shell> mysql db1 < dump.sql

如果生成的dump文件内不含有 `CREATE DATABASE`语句，那么首先我们就要建立好库，再使用dump文件。

	mysql> create database db1;
	mysql> use db1;
	mysql> source dump.sql;

### 用mysqldump生成格式化文本转储

如果在使用`mysql`的时候指定`--tab=dir_name`选项，那么会把`dir_name`指定的目录作为输出目录，每个表会在这个目录下生成两个文件。对于一个表`t1`，会生成`t1.txt`和`t1.sql`两个文件，`t1.sql`包含`CREATE TABLE`语句，`t1.txt`包含数据，每个记录一行。

比如我们要转储 数据库 db1到 /tmp目录：

	shell> mysqldump --tab=/tmp db1

`.txt`文件是由mysqld进行写入的，所以其被运行`mysqld`的账户所有用。实际上`mysqld`是使用`SELECT ... INTO OUTFILE`语句进行写入文件的，所以必须对目录具有这个权限，而且，`t1.txt`必须是不存在的。`t1.sql`同样。

最好只在本地使用`--tab`选项。如果在客户端指定远程服务器上的`--tab`选项，那么`--tab`后面的目录在客户端和服务端都必须存在。`.txt`文件只会写出到远程服务器，而`.sql`文件在本地和远程服务器都会生成。

对于`mysqldump --tab`，默认情况下每个记录一行，字段间用`tab`键进行分割，`\n`被作为换成符，对值不会以引号进行引用。这和`SELECT ... INTO OUTFILE`类似。

如果想以一种不同的格式来转储文件，有几个选项可以进行指定：

 - --fields-terminated-by=str 指定分列字符 (default: tab).
 - --fields-enclosed-by=char 指定列值被什么符号包围 (default: no character).  
 - --fields-optionally-enclosed-by=char 指定非数值被什么所包围 (default: no character).  
 - --fields-escaped-by=char 反引特殊字符的字符 (default: no escaping).  
 - --lines-terminated-by=str 指定换行符 (default: newline).

有时候，我们指定的符号，可能会被`shell`当作特殊字符对待，这样我们就要对它进行引用。或者， 这几个选项我们可以用16进制代码来指定。现在假设我们想要这个列值被双引号`"`所包围，但这个符号一般会被`shell`给特殊对待，所以我们必须对他进行引用。

	--fields-enclosed-by='"'

在任何平台上，我们可以用16进制值：

	--fields-enclosed-by=0x22

同时指定几个选项很正常。比如，要用逗号分割列，用`\r\n`进行换行，用`"`包围值：
	
	shell> mysqldump --tab=/tmp --fields-terminated-by=,\
	       --fields-enclosed-by='"' --lines-terminated-by=0x0d0a db1

不过在使用这个文件的时候，我们需要对`mysql`指定同样的选项哦。

### 使用文本格式化转储
因为文本化转储使用的是两个文件分别存储了表结构和数据，所以我们也需要进行两步来使用他。

	shell> mysql db1 < t1.sql
	shell> mysqlimport db1 < t1.txt

也可以在`mysql`客户端内用`LOAD DATA INFILE`语句来使用：

	mysql> use db1;
	mysql> LOAD DATA INFILE 't1.txt' INTO TABLE t1;

如果在生成转储的时候指定了不一样的格式，那么我们这时候也要指定同样的格式：

	shell> mysqlimport --fields-terminated-by=,\
	     --fields-enclosed-by='"' --lines-terminated-by=0x0d0a db1 t1.txt

或者：

	mysql> USE db1;
	mysql> LOAD DATA INFILE 't1.txt' INTO TABLE t1
		    -> FIELDS TERMINATED BY ',' FIELDS ENCLOSED BY '"'
		    -> LINES TERMINATED BY '\r\n';
### mysqldump 建议

这一节来解决一些使用`mysqldump`的问题：
 - 如何复制一个数据库
 - 如何复制一个数据库到另一台服务器
 - 如何转储程序（触发器、过程、函数、事件）
 - 如何分别转储定义和数据

**复制一个数据库管**
```
	shell> mysqldump db1 > dump.sql
	shell> mysqladmin create db2
	shell> mysql db2 < dump.sql
```
> 这里就不要使用`--databases`选项，这样的话转储文件内的`USE`语句会将包含`USE db1`语句，将不会把数据导入到`db2`。

**复制数据库到另外一个服务器**
Server1:

	shell> mysqldump --databases db1 > dump.sql

把dump.sql复制到Server2，执行:

	shell> mysql < dump.sql

`--databases`选项的使用会让`dump.sql`文件内包含`USE db1`语句和`CREATE DATABASE`语句，如果想把数据导入一个不同名字的库，那么，不要指定`--databases`选项。

Server1:

	shell> mysqldump db1 > dump.sql

Server2:
	
	shell> mysqladmin create db2;
	shell> mysql db2 < dump.sql

**转储存储程序**
几个选项来帮助`mysqldump`转储程序（过程、函数、事件、触发器等）。
 - --events：事件
 - --routines：过程和函数
 - --triggers：触发器

`--triggers`选项是默认开启的，其他两个则不是，需要明确指定。如果不想使用这三个选项，那么`--skip-events, --skip-routines, --skip-triggers`可以帮助你。

**分开存储定义好数据**
`--no-data， -D`选项告诉`mysqldump`不要存储数据，所以其转储文件将只包含表的创建。对应的`--no-create-info`就会让`mysqldump`不要生成创建表语句，将只包含数据。

下面我们假设想要将test库的表好数据单独导出：

	shell> mysqldump --no-data test > dump-defs.sql
	shell> mysqldump --no-create-info test > dump-data.sql

对于一个只有定义的导出，指定`--events, --routines`将会把过程、函数、事件一起导出：

	shell> mysqldump --no-data --routines --events test > dump-defs.sql

**用MySQLdump来测试升级兼容性**
在准备升级mysql版本的时候，在一个单独的测试服务器看上安装新版本的服务器，再把现在的库给导出过去进行测试是否正常。
在生产服务器上：

	shell> mysqldump --all-databases --no-data --routines --events > dump-defs.sql

在升级服务器上：

	shell> mysql < dump-defs.sql
因为dump文件只包含定义，不包含数据，所以这个过程是非常快速的。这样就可以在不等待长时间数据载入操作就能测试兼容性。在过程中仔细观察出现的错误或者警告。

如果一切正常从，那么把数据给导出，然后导入新服务器：

生产服务器上：

	shell> mysqldump --all-databases --no-create-info > dump-data.sql

升级服务器上：

	shell> mysql < dump-data.sql

然后检查一下表内容和一些其他测试就OK了。

# 时间点（增量）恢复（使用二进制日志）

事件点恢复一般是在一个完整备份的恢复后，利用二进制日至来进行数据变化的重放。
> 我们这里用`mysql`客户端来处理`mysqlbinlog`解析日志后的输出。但如果二进制日至包括 `\0`(null)字符，mysql只有在指定了`--binary-mode`选项是才能正常工作。

时间点恢复基于以下几个规则：
 - 时间点恢复的数据源是在某要完整备份后二进制日至所做的增量备份。因此，服务器必须以`--log-bin`选项启动来启动二进制日志。
 为了从二进制日志恢复数据，我们必须知道当前二进制的名字和位置（坐标）。默认情况下，服务器在数据目录生成二进制日志，但可以用`--log-bin=`选项来指定一个不一样的位置。
 想观察二进制日志列表，用以下语句：

	mysql> SHOW BINARY LOGS;

想知道当前使用的日志是哪个，用下面这个语句：

	mysql> SHOW MASTER STATUS;

 - `mysqlbinlog`程序将二进制日志内的事件变化解析为文本格式以方便识别和使用。同时其也含有一系列选项可以根据事件时间或者位置来进行范围选择。**4.6.7节 mysqlbinlog - 处理二进制日志工具**
 - 执行二进制日志内的事件变化相当于事件的重放。为了这样，将`mysqlbinlog`的输出传递给`mysql`客户端的输入 。

	shell> mysqlbinlog binlog_files | mysql -u root -p 
	
 - 当你想在某一个时间或者位置进行数据恢复的时候，看一下binlog的内容是非常有用的。

	shell> mysqlbinlog binlog_files | more

或者将输出重定向到一个文件：

	shell> mysqlbinlog binlog_files > tmpfile
	shell> ... edit tmpfile ...

把输出重定向到一个文件，这时我们就可以在文件内删除特定的事件或者内容，然后再用`mysql`导入，这是非常有用的。

	shell> mysql -u root -p < tmpfile

如果不止一个二进制日志文件需要处理，安全的办法是在一个连接内进行处理，而不是分开处理。下面就是一个不怎么安全的方法：

	shell> mysqlbinlog binlog.000001 | mysql -u root -p # DANGER!!
	shell> mysqlbinlog binlog.000002 | mysql -u root -p # DANGER!!`

这所以这样在两个连接内处理不安全的原因，是因为第一个连接可能包含一个`CREATE TEMPORARY TABLE`语句，然后第二个连接可能会需要这个临时表。当第一个连接关闭的时候，mysql进程就会删除临时表。这样，第二个连接要使用这个临时表的时候就会出现**unknown table**。

下面这种方法才是推荐的:

	shell> mysqlbinlog binlog.000001 binlog.000002 | mysql -u root -p

或者将所有的二进制日志解析输出到一个单独的文件：

	shell> mysqlbinlog binlog.000001 >  /tmp/statements.sql
	shell> mysqlbinlog binlog.000002 >> /tmp/statements.sql
	shell> mysql -u root -p -e "source /tmp/statements.sql"

###  利用事件时间来进行恢复

为了指定开始时间和结束时间，用`DATETIME`格式为`mysqlbinlog`指定`--start-time`和`--end-time`选项。假如我们在2005年4月20日的上午10点删除了一个大表。我们可以这样来进行恢复：

	shell> mysqlbinlog --stop-datetime="2005-04-20 9:59:59" \
	         /var/log/mysql/bin.123456 | mysql -u root -p


这会将数据恢复到`--stop-time`指定的时间，但如果我们是在之后某几个小时内进行恢复的话，之后的数据也需要进行恢复进来。

	shell> mysqlbinlog --start-datetime="2005-04-20 10:01:00" \
	         /var/log/mysql/bin.123456 | mysql -u root -p


如此，就跳过了我们执行的那个删除表的语句了。

有的时候，我们忘记了在什么时间执行了错误的操作，我们可以把将二进制日志解析后进行观察。
通过跳过指定时间来排除要执行重放的语句并不一定工作得很好，因为有可能在那时间内执行了多个操作。

### 利用事件位置进行恢复
`--start-position, --stop-position`选项可以指定开始和结束的位置。这样能更精确的控制想要执行和不执行的语句，这在对于相同时间内执行多个语句是非常有用的，因为位置是被串行化进行记录的。我们可以通过指定时间解析二进制日志后，查看那些错误操作的语句位置，然后跳过它：

	shell> mysqlbinlog --start-datetime="2005-04-20 9:55:00" \
	        --stop-datetime="2005-04-20 10:05:00" \
	        /var/log/mysql/bin.123456 > /tmp/mysql_restore.sql
打开mysql_restory.sql文件，找到我们执行错误语句的位置，然后下面的两个例子将会跳过`368312`到`368315`位置之间的语句。

	shell> mysqlbinlog --stop-position=368312 /var/log/mysql/bin.123456 \
	     | mysql -u root -p
	
	shell> mysqlbinlog --start-position=368315 /var/log/mysql/bin.123456 \
	          | mysql -u root -p

# MyISAM表的维护及崩溃恢复
这一节讨论用`myisamchk`程序来修复`MyISAM`表。对于基本知识的话，看一下**4.6.3 myisamchk - MyISAM表维护工具**。其他表修复信息可以查看**2.11.4 重建或修复表或索引**。

可以用`myisamchk`来修复、检查、优化数据库表。接下来的章节讨论了这几个问题和怎么样设置一个表维护调度。关于怎么样使用`myisamchk`来获取表的信息，参考**4.6.3.5 用myisamchk获取表信息**。

尽管`myisamchk`来修复表是安全的，但是做一个备份，也是非常必要的操作。在其他对表进行操作的时候一样这样考虑。

`myisamchk`操作影响索引的话，会导致用`full-text`参数来重建`FULLTEXT`索引，这与MYSQL使用的值不兼容。为了避免这个问题，参考**4.6.3.1 myisamchk 一般建议**。

`MyISAM`表可以通过SQL语句的执行来达到`myisamchk`同样的结果：

 - 用`CHECK TABLE`来检查`MyISAM`表
 - 用`REPAIR TABLE`来修复`MyISAM`表
 - 用`OPTIMIZE TABLE`来优化`MyISAM`表
 - 用`ANALYZE TABLE`来分析`MyISAM`表
关于这些语句的信息，查看**13.7.2 表维护语句**。
这些语句其实可以通过`mysqlcheck`语句来执行。使用这个的语句就是所有的事情都由`myisamchk`来执行。在使用`myisamchk`的时候，必须确定服务器没有使用对应的表，以避免两者之间的交互。

## 用myisamchk来进行崩溃恢复
这一节描述了怎么样检查和处理数据错误。如果你的数据库经常出现问题，应该尝试着找出原因。**B.5.3.3 为什么MYSQL总是崩溃**。
对于为什么`MyISAM`表会出现问题的解释，看一下**15.3.4 MyISAM表问题**。

当以禁止外部锁定方式运行`mysqld`（默认就是这样）时，在`mysqld`使用同一个表的时候执行`myisamchk`是不可靠的。如果你能确定在使用`myisamchk`之间没有人会使用那个表，你只需要在执行命令之前执行`mysqladmin flush-tables`。如果无法保证，那么必须先停止`mysqld`再执行检查。当在执行`myisamchk`的时候`mysqld`正在执行更新，那么你就会得到一个警告会产生冲突。

如果`mysqld`以外部锁启用的形式运行，可以在任何时候使用`myisamchk`。在这样的情况下，如果`mysqld`试图更新表，它必须等待`myisamchk`执行完毕。

在用`myisamchk`修复或者优化表的时候，必须总是保证`mysqld`不会使用这个表。在没有停止`mysqld`的情况下，必须先执行`mysqladmin flush-tables`。如果`myisamchk`和`mysqld`同时访问表的会话出现问题。

当进行崩溃恢复的时候，必须要清楚，一个`MyISAM`表包括三个文件。
 - tbl_name.frm 定义文件
 - tbl_name.MYD 数据文件
 - tbl_name.MYI 索引文件
多数情况下都是MYD和MYI文件产生问题。
`myisamchk`通过逐行复制**.MYD**文件创建副本，创建完成的时候就会删除原来的MYD文件，并重新命名新建的文件。如果加上`--quick`选项，`myisamchk`假设`.MYD`文件是正常的，只是产生一个新的索引文件。这是安全的，因为`myisamchk`会自动检查`.MYD`文件是否有问题，有问题会自动停止。可以指定`--quick`两次，这样，在出现某些错误（重复键错误）的时候它并不会停止，而是会试图修改`。MYD`文件进行解决。一般只有在拥有很小的空闲磁盘空间的情况下会加上两个`--quick`，不过在这样的情况下，最好先做好表的备份。

## 如何检查MyISAM表的错误
以下命令：
 - myisamchk tbl_name
 这会找出99.99%的错误。不能找出的一般就是只是**.MYD**文件的错误。一般来说，会不带任何选项，或者`-s`（slient）选项运行`myisamchk`。
 - myisamchk -m tbl_name
 这会找出99.999%的错误。首先会检查所有的索引错误，然后再读取所有行。会对所有的键值计算一个校验和并与索引树内的进行对比。
 - myisamchk -e tbl_name
This does a complete and thorough check of all data (-e means “extended check”). It does a check-read of every key for each row to verify that they indeed point to the correct row. This may take a long time for a large table that has many indexes. Normally, myisamchk stops after the first error it finds. If you want to obtain more information, you can add the -v (verbose) option. This causes myisamchk to keep going, up through a maximum of 20 errors.
 - myisamchk -e -i tbl_name
 与前面类似，不过`-i`选项会打印出附加的状态信息。

 多数情况下，简单的一个不带选项参数的命令就够了。
## 如何修复一个MyISAM表
本节讨论如何在`MyISAM`表上使用`myisamchk`。我们依然可以使用SQL语句来`CHECK TABLE`和`REPAIR TABLE`来检查和修复表

故障一般会有类似的错误：
 - tbl_name.frm 锁定无法改变
 - 找不到tbl_name.MYI(错误码:nnn)
 - 非期望的文件结尾
 - 记录文件已经崩溃
 - 从表控制处获得`nnn`错误。
为了获得关于错误更多的信息，执行`perror nnn`。下面的例子展示了大多数的错误：

	shell> perror 126 127 132 134 135 136 141 144 145
	MySQL error code 126 = Index file is crashed
	MySQL error code 127 = Record-file is crashed
	MySQL error code 132 = Old database file
	MySQL error code 134 = Record was already deleted (or record file crashed)
	MySQL error code 135 = No more room in record file
	MySQL error code 136 = No more room in index file
	MySQL error code 141 = Duplicate unique key or constraint on write or update
	MySQL error code 144 = Table is crashed and last repair failed
	MySQL error code 145 = Table was marked as crashed and should be repaired

135、136通过一个简单的修复就可以解决。这种情况下，必须用`ALTER TABLE`来增加`MAX_ROWS`和`AVG_ROW_LENGTH`表选项：

	ALTER TABLE tbl_name MAX_ROWS=xxx AVG_ROW_LENGTH=yyy;

如果你不知道当前表的选项值，使用`SHOW CREATE TABLE`语句。
对于其他错误，你必须修复你的表。`myisamchk`可以检查并修复大多数出现的错误。

修复过程有四步。在操作之前，先切换到数据目录，并检查对文件的权限。在UNIX必须保证运行`mysqld`的用户具有渡权限，如果要进行修改操作的话，还必须具有写权限。

这节来讨论检查失败，或者想要使用一些`myisamchk`提供的扩展选项。

如果想要开始修复一个表，首先要停止`mysqld`。
> 你对远程服务器执行`mysqladmin shutdown`的时候，服务器还有一段时间是可用的，必须等待正在执行的语句完毕，数据刷新到磁盘才会关闭。

Stage 1: 检查表
用 `myisamchk *.MYI` 或 `myisamchk -e *.MYI`如果世界足够的话。用`-s`选项来避免不必要的信息。
如果`mysqld`说停止的，那么用`--update-state`来告诉`myisamchk`表明表已经检查过了。
只需要修复`myisamchk`报错的表，进入Stage 2。
如果遇到了未知的错误（如内存溢出问题），或者`myisamchk`崩溃，进入Stage 3。

Stage 2:简单的安全修复
首先，尝试`myisamchk -r q tbl_name`（-r -q是快速恢复模式的意思）。这会尝试修复索引文件，不会创建数据文件。如果数据文件包含了其应该包含的所有数据以及在数据文件中当前未知的删除连接点？这将会工作，表就会被修复。否则的话按照以下过程处理：

 1. 备份数据文件。（*.MYD）
 2. 用`myisamchk -r tbl_name`命令。这将会删除错误的行，并重建索引文件。
 3. 如果第二部失败了。使用`myisamchk --save-recover tbl_name`。安全恢复模式使用一种老的恢复方法来对付几种情况，常规恢复方法不会处理。（但是更慢）
 > 如果想要修复速度更快，设置`sort_buffer_size, key_buffer_size`分别为你内存的25%。
 遇到一些意外的问题，进入Stage 3。

 Stage 3:困难的修复
 到打这一步的可能是索引文件的开始16KB已经被销毁或者出现了错误信息，或者索引文件丢失。这样的情况下，创建一个新的索引文件是必须的。按下操作：

  1. 备份数据文件。（*.MYD）
  2. 用表描述文件来创建新的（空的）数据文件好索引文件。

	shell> mysql db_name
	mysql> SET autocommit=1;
	mysql> TRUNCATE TABLE tbl_name;
	mysql> quit
  3. 把备份的数据文件覆盖到现在新建的空的数据文件。
 > 在使用复制服务器的时候，要首先停止复制服务器。因为这些属于文件系统的操作，mysql并不会记录到日志。

 回到 Stage 2。
 同样可以使用`REPAIR TABLE tbl_name USE_FRM`语句，这与上述操作一样。

 Stage 4：非常困难的修复
 到达这一步的话，唯一的可能就是连*.frm文件都崩溃了。这应该从不会发生，因为当表创建后，这个文件就不会再改变。

  1. 从备份恢复描述文件，然后回到Stage 3。也可以恢复索引文件然后会开采Stage 2。稍后，你就可以使用`myisamchk -r`。
  2. 如果没有备份描述文件，但是确切的知道表是怎么样创建的，在另外一个数据库创建同样的表。删除新建的数据文件，把新建的*.MYI好*.frm文件复制到崩溃的数据库。然后回到 Stage2，尝试重建索引文件。

## MyISAM表优化
为了整理碎片行好避免磁盘空间的浪费，运行：

	shell> myisamchk -r tbl_name

可以用`OPTIMIZE TABLE`语句达到相同的目的。`OPTIMIZE TABLE`进行一个表修复和键分析，同事排序索引树以便搜索更快。
`myisamchk`提供很多选项用来提高表的性能：

 - --analyze or -a: Perform key distribution analysis. This improves join performance by enabling the join optimizer to better choose the order in which to join the tables and which indexes it should use.

 - --sort-index or -S: Sort the index blocks. This optimizes seeks and makes table scans that use indexes faster.

 - --sort-records=index_num or -R index_num: Sort data rows according to a given index. This makes your data much more localized and may speed up range-based SELECT and ORDER BY operations that use this index.

    
## 设置一个MyISAM表维护调度计划
进行常规的检查儿不是在出现问题的时候在检查是个非常好的主意。检查和修复表的一个办法是使用`REPAIR TABLE`好`CHECK TABLE`语句。

另外一个方式就是使用`myisamchk`程序。为了维护的目的，可以用`myisamchk -s`，这会只在出现错误的时候打印出来信息。
进行自动的MyISAM表检查也是非常棒的。比如说，当一个服务器在更新时发生了重启，你必须要检查所有的表是不是收到了影响。如果要让服务器自动的检查`MyISAM`表，一`--myisam-recover-options`选项。

也可以设置一个cron任务来定时的进行检查表：

	35 0 * * 0 /path/to/myisamchk --fast --slient /pat/to/datadir/*/*.MYI

通常情况下，MYSQL表需要更少的维护工作。如果对MyISAM表进行了许多动态行更新（具有varchar, BLOB, TEXT列的表）或者表已经进行了很多删除工作，你可能想要及时进行碎片整理或者释放空间。可以用`OPTIMIZE TABLE`来达成目的。当然，如果你能停止`mysqld`一会儿，切换到数据目录，然后执行下面的命令：

	shell> myisamchk -r -s --sort-index --myisam_sort_buffer_size=16M */*.MYI
