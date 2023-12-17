---
title: MySQL备份介绍及实施
date: 2017-12-07 14:12:19
updated: 2017-12-07 15:16:19
categories: 数据库
tags: 
  - Mysql
---
MySQL 备份的方式
# 为什么需要备份
因为谁都无法保证，我们的存储设备，或者电源，或者是硬件等会在什么时候出问题，所以做一个备份，将有助于我们的数据的安全性的提高。如果你不在乎怎么给老板解释数据永久丢失无法找回，那么备份不备份并不重要了。
# 数据的存储方式
数据是以文件的格式存放在磁盘上，这些文件，由 **mysqld** 程序进行读取、写入等等操作。当然，我们也可以用文件系统上的命令来进行操作它，比如：复制、删除、修改等等操作，不过通常情况下是不推荐也不会这样去做的。由此而来，备份也多了另外一种方式。
# 备份的方式
## 逻辑备份与物理备份
从操作对象上来讲，我们可以对数据库最终的底层文件进行复制并转移到其他位置来进行备份；或者，我们可以用查询语句的方式，得到我们想要的结果，然后在需要的时候进行插入到新的位置来进行备份。这就是所谓的：** 基于语句的备份（逻辑备份）和基于文件[块]的备份（物理备份）**。
## 完整备份与增量备份
有的时候，我们需要目标数据库所有的数据集合，而某些时候，我们只需要备份目标数据库从某一时间开始来变化过的数据。这就称为 **完整备份** 和** 增量备份**。
## 可能忽略的问题
对于 **mysqld** 而言，底层数据、存储引擎、查询解析及缓存从上而下抽象分层。试想一下，我们采用**逻辑备份** 与 **物理备份** 的时候可能会出现什么问题。
当我们需要查询读取数据的时候，存储引擎会去读取磁盘文件上的数据，这没有什么问题；但当我们进行更新查询的时候，问题就出现了，如果我们在进行文件复制进行备份的时候，有人在对数据进行更新，那很明显，就会引起数据的不一致性产生；同样，当我们在进行逻辑备份的查询更新的时候，也会有人在对数据更新，这同样会产生不一致。
就我理解而言，所谓的一致，是指在开始备份至备份完成这个时间窗口内所看到的数据没有产生变化。
## 考量
我们在备份的时候，有的时候，可能想要快速的备份，而有的时候，我们则需要考虑一下兼容性。所以在**逻辑备份**和**物理备份**间会进行权衡考虑。
而当我们是不是要停止程序以中断服务的时候，就考虑采用**热备份**还是**冷备份**。
当数据太大的时候，每次都采用**完整备份**并不是一个明智的选择，经常性**增量备份**配合偶尔的**完整备份**是一个不错的执行方式。
# 对数据一致性的保证
为了保证我们备份数据的一致性，我们必须保证在备份时间窗口内没有数据更新 或者 我们看到的数据始终是不变的。
已经有一些方法来达成这个目的。
## flush配合读锁表
	mysql> flush tables with read lock;
> 这个语句会将缓存刷新到磁盘，关闭所有打开表，读锁定所有表。

1.建立一个测试表。

	mysql> create table ssdd.tbl( id int auto_increment, name varchar(20), primary key (id)) engine=MyISAM;
	mysql> insert into ssdd.tbl(name) values('angel');

我们不妨自己测试一下，首先我对 ssdd.tbl 表进行查询一下，这会导致 **mysqld** 打开这个表。

	mysql> select id from ssdd.tbl;
	+----+
	| id |
	+----+
	|  1 |
	+----+
2.查看打开文件
然后我们用 **Linux的 lsof **命令看一下 打开的文件情况。

	[root@VM_0_6_centos ~]# lsof | grep tbl
	COMMAND     PID    USER   FD      TYPE             DEVICE SIZE/OFF       NODE NAME
	mysqld    14634   mysql   16u      REG              252,1     2048     368758 /var/lib/mysql/ssdd/tbl.MYI
	mysqld    14634   mysql   17u      REG              252,1       20     368759 /var/lib/mysql/ssdd/tbl.MYD
3.锁

	mysql> flush tables with read lock;
4.查看文件打开情况


	[root@VM_0_6_centos ~]# lsof | grep tbl
	[root@VM_0_6_centos ~]#
输出表明，这表文件已经被关闭了。
5.查询数据（另外一个终端）

	mysql> select id from ssdd.tbl;
	+----+
	| id |
	+----+
	|  1 |
	+----+
6.更新一下数据试试

	mysql> update ssdd.tbl set id=id+1;
你会发现，卡住了，是的，因为表是被读锁定的。那么我们下一步解锁。

7.解锁

	mysql> unlock tables;
这样**第6步**更新才会成功。
**对于MyISAM**表，**mysqlhotcopy**采用的就是这种方式进行备份。当然，我们可以自己手动执行给表上锁被刷新语句后，自行进行复制文件进行备份。
## 对于事务表
对于InnoDB，最好看一下其简单的概念。[MySQL架构简介](/MySQL/InnoDB-Architecture.html)
常规概念上来说，**MyISAM**与**InnoDB**引擎，一个不支持事务，一个支持事务。但在数据的存储方式上，也有不同。
**MyISAM**表的数据保存在三个文件内 tbl.{frm,MYD,MYI}
**InnoDB**表的数据并存储在由多个数据文件构成的**表空间**内，同时，重做日志会存储在重做日志文件内。**数据文件**与**重做文件**是两个非常重要的概念，对于**InnoDB**表而言。**注意：二进制日志与重做日志是不同滴**。
我们可以通过实例来查看一下其中的一些细节。
1.创建一个 innodb 表。

	create table ssdd.tbl2 ( a int auto_increment, b char(20), primary key (a)) engine=innodb;
2.查看表的存储文件

	shell> ls -1 /var/lib/mysql/ssdd	--/var/lib/mysql/ssdd 是数据库目录
	db.opt
	t_ability.frm
	t_ability.MYD
	t_ability.MYI
	tbl2.frm
	tbl.frm
	tbl.MYD
	tbl.MYI
我们只看到了 __tbl2.frm__ 也就是表和列定义文件，而并没有 如同 **MyISAM**表那样的数据文件和索引文件，这都存储于表 **InnoDB**的表空间内，默认情况下，是在*datadir* 中的 *ibdata1* 文件。
有人可能会反对，说**innodb\_file\_per\_table** 配置可以每个表使用不同的 表空间文件，这当然是可以的。

>对那些想把特定表格移到分离物理磁盘的用户，或者那些希望快速恢复单个表的备份而无须打断其余InnoDB表的使用的用户，使用多表空间会是有益的。

但我们要意识到，**innodb\_file\_per\_table**选项只会影响表的创建。也就是说，对于已在共享表空间创建的表，不会受到这个选项的影响；而如果在创建表后，关闭这个选项，已创建的使用单独表空间的表，也不会受到影响。
共享表空间，依然是需要的，因为**InnoDB**把内部数据词典和未作日志放在这个文件中。

配置**innodb\_file\_per\_table**选项，并重启服务器,然后建立一个表。

	create table ssdd.tbl3 ( a int auto_increment, b char(20), primary key (a)) engine=innodb;
查看目录:

	[root@VM_0_6_centos mysql]# ls -1 /var/lib/mysql/ssdd
	db.opt
	t_ability.frm
	t_ability.MYD
	t_ability.MYI
	tbl2.frm
	tbl3.frm
	tbl3.ibd
	tbl.frm
	tbl.MYD
	tbl.MYI
表tbl3有了自己的.ibd表空间数据文件。
但是，即使是这样，我们在进行物理备份的时候依然会很头疼。因为我们不仅要备份表的数据文件，还要备份一些数据字典文件（位于系统共享表空间内），还有重做日志（ib_logfile0/1）等等。
而且，从概念上讲，磁盘上的数据文件与InnoDB引擎在内存中的缓存池、二次缓存的数据并不一定是一致的。在事务已提交到重做日志，但还未更新到数据文件之间的时间窗口会产生很大的问题。
