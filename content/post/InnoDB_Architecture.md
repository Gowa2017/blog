---
title: InnoDB的架构介绍
date: 2017-12-04 14:02:38
categories: 
  - 数据库
tags: 
  - MySQL
---
# 架构图
这是对官方文档 5.5版本第十四章的一个翻译。
先理解一下InnoDB的架构，然后对几个基本的概念有个理解，对后面的各种操作就会有比较直观的感觉，而不会感觉非常的抽象。
下面是转自网络的一个架构图。
<!-- more -->
# 14.1 InnoDB的介绍
**InnoDB**是一个在可靠性好高性能之间进行平衡后设计的一个存储引擎。从MySQL5.5开始，默认的存储引擎就从**MyISAM**转到了**InnoDB**。在没有特别指定一个默认存储引擎的情况下，用`Create Table`语句不附加`ENGINE=`语句就会创建一个**InnoDB**表。

**InnoDB**包括了在MySQL5.1中作为插件存在的 InnoDB Plugin的所有功能，加上MySQL5.5或者更高版本增加的其他新功能。
> `mysql`和`INFORMATION_SCHEMA`库这些MySQL的内部实现还使用`MyISAM`。实际上，不能把授权表改为**InnoDB**引擎。

**InnoDB的关键优势**  
- DML语句遵守ACID模型，事务的特性提交、回滚、崩溃恢复保护用户数据。参考**14.5 InnoDB和ACID模型一节**。  
- 行级别的锁定和Oracle-Style的一致性读增强了多用户并发和性能。参考**14.8 InnoDB锁定和事务模型**。  
- InnoDB会用主键对磁盘表数据进行查询优化。每个InnoDB表还有一个主键索引被称为`clustered index`，这被用来组织数据以实现主键搜索的最小化I/O。参考**14.11.2.1 Clustered and Secondary Indexes**。   
-  为了保证数据完整性，InnoDB支持**外键**约束。通过外键约束，插入、更新和删除都会被检查以保证跨表间的一致性。参考**14.11.1.6 InnoDB and FOREGIN KEY Constraints**。
table 14.1 InnoDB Storage Engine Features

|||||||
|---|---|---|---|---|---|
|Storage Limits|64TB|Transactions|Yes|Locking granularity|yes[a]|
|MVCC|yes|Geospatial data type support|yes|Geospatial indexing support|yes|
|B-tree indexes|yes|T-tree indexes|No|Hash indexes|No[b]|
|Full-text search indexes[c]|Yes|Clustered Indexes|Yes|Data caches|Yes|
|Index caches|Yes|Compressed data|Yes[d]|Encrypted data[e]|Yes|
|Cluster database support|No|Replication support[f]|Yes|Foreign key support|Yes|
|Backup/point-in-time recovert[g]|Yes|Query cache support|Yes|Update statistics for data dictionary|Yes|

>[a] InnoDB support for geospatial indexing is available in MySQL 5.7.5 and higher.  
[b] InnoDB utilizes hash indexes internally for its Adaptive Hash Index feature.  
[c] InnoDB support for FULLTEXT indexes is available in MySQL 5.6.4 and higher.  
[d] Compressed InnoDB tables require the InnoDB Barracuda file format.  
[e] Implemented in the server (via encryption functions). Data-at-rest tablespace encryption is available in MySQL 5.7 and higher.  
[f] Implemented in the server, rather than in the storage engine.  
[g] Implemented in the server, rather than in the storage engine.  

想要比较`InnoDB`和其他存储引擎的特性的话，详见**15章， Alternative Storage Engines**

**InnoDB增强和特性**
MySQL5.5版本的InnoDB引擎包括了很多性能的提高，这些内容在MySQL5.1版本只能通过安装 InnoDB Plugin来实现。最新版本的InnoDB提高了性能和可扩展性，增强了可靠性和新的扩展能力，并且更加易用。

关于更多MySQL5.5 版本中InnoDB增强和新特性，请参考：
* 1.4节 What is New in MySQL 5.5?
* Release Notes.

**更多的InnoDB信息和资源**
* InnoDB相关条目和定义，参考 **MySQL Glossary**
* 关于InnoDB存储引擎的论坛，[MySQL Forums::InnoDB](http://forums.mysql.com/list.php?22)
* InnoDB是在GBL 2.0协议下发布的。关于更多MySQL的声明，查看[https://www.mysql.com/about/legal/](https://www.mysql.com/about/legal/)

## 14.1.1 使用InnoDB表的好处
如果你正在使用**MyISAM**表但是因为技术上的原因对它并不是很放心的，那么你会发现**InnoDB**有以下的好处：
* 如果服务器因为硬件或者软件的问题崩溃，不过崩溃的时候数据库在做什么，你只需要重启服务器而不需要做更多其他工作。**InnoDB crash recovery**自动完成那些在崩溃时提交的改变，回滚未提交但在处理的改变。只需要重启然后继续工作。现在这个过程比在5.1版本及以前版本都要快了很多
* InnoDB在缓冲池内存储被访问过的数据和索引。经常使用的数据可以直接从内存获取，这会大大提高速度。在一个单一的数据库服务器上，经常会被缓冲池大的小设置为物理内存的80%。
* 如果把相关性的数据分隔到不同的表内，可以设置**外键**来强制进行相关完整性。更新或者删除数据，在其他表内的数据会被自动的更新或者删除。当在一个次表内插入一个在主表内并不对应的数据时，错误的数据将会被自动踢出。
* 如果数据在磁盘或者内存上出现损坏，一个校验和算法会在你使用数据前对你进行警告。
* 当你把你的表设置为含有一个合适的主键列的时候，这些列会被自动优化。在`WHERE，ORDER BY，GROUP BY`语句和`JOIN`内引用这些列会非常快速。
* 插入，更新，删除会被**change buffering**自动优化。InnoDB不止允许对同一个表的并发读写，还会将改变的数据线性排队到磁盘IO。
* 对于执行长时间查询的巨大表性能的提升是无限制的。当表上同样的行被多次访问，一个叫做**ADI（adaptive Hash Index）**的特性会让这个查找更快，就像数据从hash表出来的一样。
* 可以自由的把InnoDB表和其他表混用，即使是在同一个语句内。比如说，可以在一个查询内用`Join`操作来结合InnoDB表数据与`Memory`表数据。
* InnoDB表被设计来在处理大量数据的时候提高CPU的效率和更大的性能提升。
* InnoDB可以操控大量数据，即使在文件大小被限制在2GB的操作系统上。

关于在应用内可以使用的`InnoDB`调整技术，查看**8.5 优化InnoDB表**


## 14.1.2 InnoDB表的最佳实践

这节描述了使用InnoDB表时的最佳实践：
* 把每个表最常查询的列作为表的主键，如果没有主键的话设置一个**auto-increment**值。  
* 当要从多个表内通过特定的ID值获取数据时，使用`join`操作。为了提高`join`性能，对join的列定义`foregin keys`，并把这些列定位为相同的数据类型。增加外键可以保证引用的列都会被索引，这样就能提高性能。外键会将更新或者删除操作传递到所有受影响的表，同时在父表内无对应数据时，会阻止对子表的数据插入。  
* 关闭`autocommit`。一秒内提交事务上百次是对性能的巨大浪费（这收到写入到存储设备速度的限制）。  
* 将相关的**DML**操作在事务内进行分组，用`START TRANSACTION`和`COMMIT`语句包围。尽管你不想太过频繁提交，你同样也不希望在一个巨大包括`INSERT,UPDATE,DELETE`的语句运行几个小时而不提交。  
* 不要使用`LOCK TABLES`语句。InnoDB可以控制多个会话同时读写同样的表，不用担心可靠性和性能的问题。如果要对某些行获得读占的写权限，使用`SELECT ... FOR UPDATE`来锁定你打算操作的行。  
* 开启`innodb_file_per_table`让每个InnoDB把自己的数据和索引存储在单独的文件内，而不是放在共享的表空间内。这个特性设置对于使用某些特性是必须的，比如 表压缩 和 快速 截断。  
* 评估一下你的表和访问模式是否能从 InnoDB表的压缩特性（ROW_FORMAT=COMPRESSED）上受益。可以压缩 InnoDB表却不用担心读/写性能。  
* 为了避免你在使用`CREATE TABLE`语句的时候指定`ENGINE=`语句而使用不同的存储引擎，给服务器启动添加`--sql_mode=NO_ENGINE_SUBSTITUTION`选项

## 14.1.3 检查 InnoDB可用性
为了确定你的服务器是否支持InnoDB：
* 执行命令`SHOW ENGINE`;然后查看所有的MySQL存储引擎。查找`InnoDB`行看是否有`DEFAULT`字样，。或者，可以查询`INFORMATION_SCHEMA ENGINES`表。（5.5版本以后，INNODB已经是默认的存储引擎，只有在非常特殊的情况下才不是）  
* 执行`SHOW VARIABLES LIKE 'have_innodb';`确认InnoDB可用。  
* 如果InnoDB不存在，那么你所获得的版本不支持InnoDB。需要重新获得另外一个支持的版本。  
* 如果InnoDB是禁止的，回到启动选项文件，然后去掉任何`skip-innodb`选项。  

## 14.1.4 向上和向下兼容性

在MYSQL 5.5 介绍了一个使用 InnoDB表压缩的能力，还有使用这个能力必须使用的新的行格式，叫做`Barracuda`。以前的那种格式被称做`Antelope`，不支持表压缩能力，不过支持其他能力。

## 14.1.5 测试和benchmarking InnoDb

在完成升级到MySQL5.5之前，应该先试试现在的数据库是否能在InnoDB作为默认引擎的情况下工作得很好。如果要在更早一些的版本上把InnoDB设置为默认引擎，在命令行加入`--default-storage-engine=InnoDB`选项，或者在`my.cnf`文件内`[mysqld]`节加上`default-storage-engine=innodb`选项，然后重启服务器。

因为修改默认引擎只会影响新表的建立，首先确认应用程序已经全部安装完毕。然后测试一下数据的载入，修改和查询工作OK。如果表依靠某些MyISAM特定的特性，你就会收到错误；对建立表`CREATE TABLE`加上`ENGINE=MyISAM`语句来避免这样的错误。（比如，需要全文搜索的表必须使用MyISAM而不是InnoDB）

如果你不是很确定到底用哪一个引擎，只是想看一下特定的表在InnoDB下工作得怎么样，执行命令`ALTER TABLE table_name ENGINE=InnoDB`.或者可以执行语句制造一个表的备份：

	CREATE TABLE InnoDB_Table (...) ENGINE=InnoDB AS SELECT * FROM MyISAM_Table;

因为InnoDB在MySQL5.5版本进行了很多性能提升，想要在特定的工作情况下来决定是否使用的话，安装MySQL5.5然后benchmark一下。

测试整个程序的所有环节，安装，高负载使用，服务器重启等等。 在服务器繁忙的时候杀死服务器进程来模拟一个电源问题，然后验证数据会在重启服务器的时候成功自动恢复。

测试任何复制设置，特别是在主从服务器间使用一个不同的MySQL版本的时候。

## 14.1.6 关闭 InnoDb
总体还是建议使用InnoDB作为默认存储引擎，从个人博客或者高端服务器都可以。如果实在不想使用的话：
* 用`--innodb=OFF`或`--skip-innodb`选项启动服务器来禁止InnoDB存储引擎。  
* 因为默认存储引擎就是InnoDB，同时还需要指定`--default-storage-engine`选项指定新的默认引擎才会成功启动。  
* 为了避免服务器在查询InnoDB相关的`information_schema`表时崩溃，还需要关闭一些其他的属性。在配置文件的`[mysqld]`节下面加上：

```
loose-innodb-trx=0  
loose-innodb-locks=0  
loose-innodb-lock-waits=0  
loose-innodb-cmp=0
loose-innodb-cmp-per-index=0
loose-innodb-cmp-per-index-reset=0
loose-innodb-cmp-reset=0
loose-innodb-cmpmem=0
loose-innodb-cmpmem-reset=0
loose-innodb-buffer-page=0
loose-innodb-buffer-page-lru=0
loose-innodb-buffer-pool-stats=0
```

# 14.2 安装InnoDB存储引擎

如果在使用MySQL 5.5以上版本，或者使用InnoDB1.1以上版本，那么没有什么需要特别做的：所有东西都已经作为MySQL源代码和二进制版本的一部分配置好了。这和早期的InnoDB Plugin不同。

从MySQL 5.1.38开始，InnoDB就已经包含在里面了。

为了更好的利用现在InnoDB的特性，强烈建议在配置文件内加上下面的配置：

```
innodb_file_per_table=1
innodb_file_format=barracuda
innodb_strict_mode=1
```

# 14.5 InnoDb和ACID模型

ACID模型是一系列的数据库设计规则，这些规则强调了商业数据和极端环境下对可靠性各方面。MySQL包含如InnoDB存储引擎这样的组件来符合ACID模型，用来保证在意外情况如软件崩溃、硬件问题发生的时候数据不会损坏，查询结果不会混乱。如果你依赖于ACID兼容特性，不需要重新造一个一致性检查和崩溃恢复方法的轮子。假入你有一些附家的软件保护，硬件冗余，或者一个可以容忍少量数据丢失或不一致的应用，你可以调整MySQL在ACID可靠性和提高性能和吞吐量上进行平衡。

接下来的章节讨论了MySQL特性，实际上就是InnoDB存储引擎，与ACID模型分类的交互。

* **A**：原子性（atomicity）
* **C**：一致性（consistency）
* **I**：隔离性（isolation)
* **D**：持久性（durability）

**Atomicity（原子性）**  
ACID模型的原子性与InnoDB的**事务**联系。相关MySQL特性包括：  
* 自动提交设置（autocommit)  
* `COMMIT`语句  
* `ROLLBACK`语句  
* 从`INFORMATION_SCHEMA`表上操作数据。  

**Consistency（一致性）**  
ACID的一致性与InnoDB保护数据崩溃联系。相关特性包括：  
* InnoDB **double buffer**  
* InnoDb **crash recovery**  

**Isolation（隔离性）**  
ACID的隔离性主要与InnoDB的事务联系，实际上是每个事务的**isolation level（隔离级别）**。相关特性包括：  
* 自动提交设置（Autocommit）  
* `SET ISOLATION LEVEL`语句  
* InnoDB锁的最低级别。在调整性能期间，可以通过`INFORMATION_SCHEMA`表看到细节信息。

**Durability（持久性）**  
ACID的持久性由MySQL的软件与实际硬件配置来支持。根据CPU、网络、存储设备的不同，这有多种可能，所以说对这个持久性提供明确的指南是最复杂的一部分。（这个指南可能还涉及购买“新硬件”）。相关特性包括：  
* InnoDB **doubule buffer**，通过`innodb_doublewrite`配置进行开关。  
* `innodb_flush_log_at_trx_commit`配置选项  
* `sync_binlog`配置选项  
* `innodb_file_per_table` 配置选项  
* 将缓存写入存储设备，如磁盘，SSD，或者RAID阵列。  
* 存储设备电源备份  
* 运行MySQL的操作系统，实际上是要其支持`fsync()`系统调用。  
* 不间断电源（UPS）保护所有运行MySQL的服务器和存储设备。  
* 备份策略，比如经常性和备份类型，以及备份保留时间  

# 14.6 InnoDB 多版本控制

InnoDB是一个多版本的存储引擎：它会保留改变行数据的老版本信息，用以支持事务性的特性，如一致性和回滚。这些信息保存在表空间一个叫**回滚段（rollback segment）**的数据结构中。InnoDB用回滚段内的信息来对事务的回滚进行撤销操作。同样，也用这里面的信息来建立一个早期版本以支持**一致性读**。

内部实现，InnoDB对每个行增加三个字段。一个 6byte 的`DB_TRX_ID`字段来表明上一个插入或者更新这行的事务ID。一个删除操作也被当作一个更新操作，但是一个特殊的bit会被设置来表明实际是一个删除操作。同样每行也包含一个 7byte 的`DB_ROLL_PTR`字段叫做滚动指针。这个滚动指针指向一个写在回滚段的**撤销日志记录**。如果一行被更新了，这个撤销日志记录会包含将此行恢复到原来状态的所有信息。还有一个 6byte 的`DB_ROW_ID`在新行插入的时候会自动增长。如果InnoDB会自动生成一个 `clustered index`，这个索引就会包含行ID的值。否则，`DB_ROLL_ID`不会出现在任何索引内。

回滚段内的撤销日志被分为插入和更新撤销日志。插入撤销日志只会被事务回滚需要，当事务提交的时候就可以丢弃。更新撤销日志在一致性读的时候也需要，但只有在没有被InnoDB在一致性读内分配了快照的事务活跃的时候丢弃。一致性读需要更新撤销日志来建立对这行的早期版本。

正常提交事务，及时是需要一致性读的事务。否则，InnoDB不能丢弃更新撤销日志，回滚段就会越来越大，充满表空间。

回滚段内的一个撤销日志的物理大小通常会比其对应插入或更新的行小。可以用这些信息来计算回滚段需要的空间。

在InnoDB的多版本中，用SQL删除一行的时候，这一行并不会立刻物理地从数据库内删除。只有在丢弃了对应更新撤销日志时InnoDB才会物理删除行和索引。这个操作被称作`purge`，非常快。

如果在一个小的脚本内插入或者删除行，这个`purge`线程会落后，然后表就会越来越大，因为有很多`dead`行。这样的情况下，控制新行操作，并且分配更多的资源给purge线程。通过`innodb_max_purge_lag`系统变量设置。

## 多版本与二级索引

InnoDB多版本并发控制对待`二级索引(Secondary index)`和`聚簇索引(clustered index)`不同。对聚簇索引的更新是立即的，同时更新其指向撤销日志（这些日志可以用来重建早期版本记录）的隐藏系统列。跟聚簇索引不同，二级索引不会被立即更新，也不包括系统隐藏列。

当一个二级索引列被更新时，老的二级索引记录就标记为删除，新的索引被插入，然后删除标记为删除的老的索引记录。当一个二级索引记录被标记为删除或者二级索引页被一个新的事务更新的时候，InnoDB在聚簇索引内寻找记录。在聚簇索引中，这条记录的`DB_TRX_ID`被检查，如果记录是在读取事务初始化后修改的，对应版本的记录会从撤销日志内读取。

如果一个二级索引记录被标记为删除或者二级索引页被一个新的事务更新，`covering index`技术就不会被使用。InnoDB会从聚簇索引读取记录而不是从索引结构内获取值。

# 14.7 InnoDB架构
本节介绍InnoDB存储引擎的主要组件。
![MySQL架构图](/res/20171204-mysql-innodb-architecture.png)


## 14.7.1 Buffer Pool
**缓存池**被**InnoDB**在内存中存储被访问的数据和索引，这样常用数据就能直接从内存读取，而不用访问磁盘，大大的提高了速度。一个专门的数据库服务器，80%的物理内存经常会被分配来做缓存池。
对于大量的区操作来说，缓存池被分割成**页**来容纳更多的行。在实现上，是将页做成一个**页链表**；不常用的数据，采用**LRU**算法进行清理出缓存。
## 14.7.2 Change Buffer
`change buffer`是一个特殊的数据结构，在二级索引页发生改变，同时这些页不存在于缓存池的时候，就会缓存到`change buffer`。已缓存的变化（可能是由`INSERT, UPDATE, DELETE`操作引起，DML），在后期某些操作将这些页读入缓存池的时候会被合并。

与聚簇索引不同，二级索引是不唯一的，对二级索引的插入以一个相对随机的顺序进行。类似的，删除和更新将会影响索引树内并不相邻的二级索引页。合并缓存的变化会在后面进行，当受影响的页被其他操作读取到缓存池的时候，这将会避免读取二级索引页时需要的大量随机IO操作。

周期性滴，清理操作（purge）会在系统空闲或者在一个 slow shutdown的时候进行，把更新的索引页写到磁盘。这个清理操作会将一系列的索引值写入磁盘，而不是每个值都立即写到磁盘，这样会大大提高效率。  

`change buffer`的合并操作有可能会花费几个小时，当有很多二级索引页被更新影响很多行的时候。在这个时间段内，磁盘 I/O增加，然后对于需要读取磁盘数据的查询产生很大影响。`change buffer`合并也可以在一个事务提交后进行。实际上，`change buffer`合并有可能发生在服务器个人关闭或者重启后。（**14.23.2 Forcing InnoDB Recovery**）

在内存中，`change buffer`是`buffer pool`的一部分。在磁盘上，它是系统表空间的一部分，这样就可以在数据库重启之间保留已缓存的索引变化。

在`change buffer`内缓存的数据由`innodb_change_buffering`配置选项进行控制。更多信息参考**14.9.4 Configuring InnoDb Change Buffering**

**监控Change Buffer**  
以下选项对于监控 `change buffer`是可用的：
* InnoDB标准监控输出包括了change buffer的状态信息。如果要查看监控器数据，执行命令`SHOW ENGINE INNODB STATUS;`  

	mysql> SHOW ENGINE INNODB STATUS\G
	

`Change buffer`状态信息在`INSERT BUFFER AND ADAPTIVE HASH INDEX`头部后面，看起来和下面有些类似：  

```
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 276707, node heap has 1 buffer(s)
15.81 hash searches/s, 46.33 non-hash searches/s
```
更多信息，参考**14.20.3 InnoDB Standard Monitor and Lock Monitor Output**  
* `INFORMATION_SCHEMA.INNODB_BUFFER_PAGE`表提供了所有在缓存池内页的信息，包括`change buffer`索引页和bitmap页。`change buffer`页用`PAGE_TYPE`区分。`IBUF_INDEX`是`change buffer`的索引页类型，`IBUFF_BITMAP`是`change buffer`的bitmap页类型。  
> 查询`INNODB_BUFFER_PAGE`表会造成巨大的性能开销。为了避免影响性能，在一个测试的实例上运行的检查。
举例说明，你可以查询`INNODB_BUFFER_PAGE`表来决定`IBUF_INDEX`和`IBUF_BITMAP`的大概数量站总共缓存池页的百分比：

```
SELECT
(SELECT COUNT(*) FROM INFORMATION_SCHEMA.INNODB_BUFFER_PAGE
WHERE PAGE_TYPE LIKE 'IBUF%'
) AS change_buffer_pages,
(
SELECT COUNT(*)
FROM INFORMATION_SCHEMA.INNODB_BUFFER_PAGE
) AS total_pages,
(
SELECT ((change_buffer_pages/total_pages)*100)
) AS change_buffer_page_percentage;
+---------------------+-------------+-------------------------------+
| change_buffer_pages | total_pages | change_buffer_page_percentage |
+---------------------+-------------+-------------------------------+
|                  25 |        8192 |                        0.3052 |
+---------------------+-------------+-------------------------------+
```

* `Performance Schema`提供了change buffer 互斥量等待方法来监控进阶性能。想查看change buffer的使用方法，执行下面的查询：

```
mysql> SELECT * FROM performance_schema.setup_instruments
WHERE NAME LIKE '%wait/synch/mutex/innodb/ibuf%';
+-------------------------------------------------------+---------+-------+
| NAME                                                  | ENABLED | TIMED |
+-------------------------------------------------------+---------+-------+
| wait/synch/mutex/innodb/ibuf_bitmap_mutex             | YES     | YES   |
| wait/synch/mutex/innodb/ibuf_mutex                    | YES     | YES   |
| wait/synch/mutex/innodb/ibuf_pessimistic_insert_mutex | YES     | YES   |
+-------------------------------------------------------+---------+-------+

```

# 14.7.3 Adaptive Hash Index
为缓存池配置大量缓存加上配置一定的工作量，`AHI（adaptive hash index）自适应哈希索引`让InnoDB更像一个内存数据库，这样就不用牺牲事务性特性和可靠性。这个特性是被`innodb_adaptive_hash_index`选项开启的，可以用`--skip-innodb_adaptive_hash_index`在启动服务器禁止。

基于观察到的搜索模式，MySQL用索引键的前缀来建立一个hash索引。这个键的前缀可以是任何长度，可能只有部分在B-tree内的值会出现在hash索引内。哈希索引只为那些需要经常性访问的索引页建立。

如果一个表全部都在主内存内，哈希索引可以通过启动直接查询任何元素提高速度，将所有索引值设置为指针。InnoDB有一个监控索引搜索的方法。如果InnoDB注意到采用哈希索引很有好处的会，就会自动的建立。

在某些工作负载下，哈希索引查询速度的提升的价值远远超过了其需要用来监控索引查找及维护哈希索引结构的额外工作。某些时候，在高负载时，读/写锁来保护对AHI的访问是一个巨大的争议问题，比如多并发的join。用`LIKE`和`%`操作符的查询也不能从AHI受益。在不需要AHI的工作下，关闭AHI可以减少不必要的性能开销。对于一个系统AHI是否适用是很难预测的，在真实的工作负载下进行开启或关闭AHI的benchmark，得出自己想要的结果。

哈希索引总是基于表上已存在的B-tree索引。InnoDB可以用在B-tree上定义的键的任何长度前缀来建立哈希索引，这依赖于InnoDB观察到的对B-tree索引搜索模式。一个哈希索引可以是局部的，只覆盖了经常访问的那些索引页。

可以监控AHI的使用和争论在命令`SHOW ENGINE INNODB STATUS`命令的`SEMAPHORES`节。如果观察到很多线程在等待RW锁在`btr0sea.c`创建，那么，关闭AHI应该非常有用。

关于更多哈希索引的性能问题，参考**8.3.8 Comparison of B-Tree and Hash Indexes**

# 14.7.4 Redo Log Buffer
**重做日志缓存器**在内存中缓存即将被写入**redo log**的数据，用**innodb\_log\_buffer\_size**。缓存器会周期性的刷新到磁盘**redo log**文件。一个大点的缓存器允许一个大型事务提交后，在没有将**redo log**写到磁盘前就开始运行。因此，如果你有事务需要更新，插入或删除很多行，使用更大的 Redo log buffer来减少磁盘I/O。

**innodb\_flush\_log\_at\_trx\_commit**选项控制缓存器的内容怎么写到**redo log**文件。
**innodb\_flush\_log\_at\_timeout**选项控制缓存器的内容刷新周期。
# 14.7.5 System Tablespace
**系统表空间**包含了**InnoDB**的数据字典（InnoDB关联对象元数据）和doublewrite buffer, change buffer, undo logs。同时也包含用户在共享表空间内创建的表的数据和索引。多个表，可共享同系统表空间。
**系统表空间**由多个数据文件组成，默认情况下是**ibdata1**，当然，可以用**innodb\_data\_file\_path**用来设置系统表空间的数量和大小。

更多信息参考**14.9.1 Resizing The InnoDb System Tablespace**。

## 14.7.6 InnoDB Data Dictionary
**数据字典**由包含用来跟踪对象（比如表、索引、列）的元数据的系统表组成。这些元数据存储在系统表空间内。历史原因，数据字典元数据一定程序上与表元数据文件（tbl.frm）有重叠。
# 14.7.7 Doublewrite Buffer
这是表空间内的一个区域，用来存储从**缓存池**刷新过来，但还没有写到数据文件的页。只有在刷新并且写到二次写入缓存器后，InnoDB才会将数据写到数据文件。

如果在一个页写入期间发生了系统崩溃、存储系统崩溃、或者**mysqld**进程崩溃，**InnoDB**可以在恢复期间从二次写入缓存找到一个完整的数据副本。
虽然数据都要写入两次，但事实上是在多量数据写入二次缓存后，再采用**fsync()**再同步到磁盘。

默认情况下二次写入缓存是开启的，可以用**innodb_doublewrite**设置为0来关闭。
如果系统表空间数据文件部署在支持原子写入的**Fusion-io**设备上，此选项将会被自动关闭，而启用**Fusion-io**的原子写入到所有数据文件。二次写入缓存是否开启这个选项是全局的，这样，在没有使用**Fusion-io**设备的数据文件也会被关闭二次写入缓存。这个特性只支持**Fusion-io**设备。

# 14.7.8 Undo Logs
**撤销日志**是一系列每个事务的撤销日志记录（**undo log record**）的集合。一个撤销日志记录包含了如何撤销一个事务对**聚簇（clustered index）**最近作出的改变。如果其他事务想要看到原始的数据（读操作一致性的一部分），未修改的数据是从撤销日志记录中获取。撤销日志存在与撤销日志段（**undo log segments**）中，撤销日志段存在与回滚段中（**rollback segments**）。回滚存留在系统表空间、临时表空间、和撤销表空间。

从5.5.4版本之前，InnoDB支持单一回滚段（支持最大1023个并发数据修改事务，只读事务不会增加这个最大限制）。在MySQL5.5.4，单一回滚段被分割为128个回滚段，每个回滚段支持1023个并发数据修改 事务修改，这样限制就增加到了大概128k并发的数据修改事务限制。`innodb_rollback_segments`选项定义了在系统表空间内为InnoDB事务使用多少个回滚段。

每个事务被分配一个回滚段，然个少的后后面就一直使用这个段。这个增加的并发修改限制提高了伸缩性（更高的并发）和性能（更少的对于回滚段使用的竞争）。
InnoDB支持128个回滚段，其中32个保留用于针对临时表事务的非重做的回滚段。每个更新临时表的事务都会被分配两个回滚段，一个重做回滚段和非重做回滚段；只读事务只会分配非重做的回滚段，只读事务只允许修改临时表。
这样就留下96个可用的回滚段，每个支持1023个并发的数据修改事务，合计就是96K。这里假定所有事务都不会修改临时表，如果所有事务都会修改临时表，限制就会降低到32K。对于更多保留用于临时表事务的回个段信息，参考下一节。
**innodb_rollback_segments**选项定义了InnoDB使用的回滚段数。
# 14.7.9 File-Per-Table Tablespaces

这个选项允许让每个表单独存在在自己的表空间文件内。`innodb_file_per_table`可以启用这个功能。

# 14.7.10 Redo Log
**重做日志**是基于磁盘的数据结构用来在崩溃恢复的时候纠正被未完成事务修改的数据。常规操作下，重做日志对SQL语句或者底层的API请求进行编码，然后请求InnoDB表修改。因意外关闭而未完成的对数据文件的修改将会在重新启动，接受连接之前重放。在崩溃恢复的时候，重做日志担当的角色，请查看[InnoDB 恢复](https://dev.mysql.com/doc/refman/5.7/en/innodb-recovery.html)

默认情况下，重做日志文件就是数据目录中的**ib_logfile0**和**ib_logfile1**。
MySQL以环行的方式写入重做日志文件。
和其他ACID兼容的数据引擎一样，在提交事务前刷新重做日志。InnoDB采用组提交的方式，多个同时间的事务的提交请求集中到一个写请求上。

## 14.7.10.1 组提交重做日志进行刷新

InnoDB，跟其他ACID兼容的数据库引擎一样，在一个事务提交以前刷新重做日志。InnoDB使用 `group commit`功能来将多个这样的请求组合在一起以避免每个提交都需要一个刷新。这样，InnoDB就可以将多个用户的在同样时间内的多个事务在一次进行刷新，大大提高了吞吐量。
# General Tablespaces
通用表空间：用`CREATE TABLESPACE`语句创建的InnoDB表空间，可以在MySQL数据目录外。
用`create table tbl_name ... tablespace [=] tablespace_name` 或者 ` alter table tbl_name tablespace [=] tablespace_name`。
# Undo Tablespace
撤销表空间由一个或者多个包含重做日志的文件组成。**innodb_undo_tablespace**设置了InnoDB使用多少个重做表空间。**这个选项将来应该会被删除。**
# Temporary Tablespace
临时表空间用来存储非压缩的InnoDB临时表和相关对象。
**innodb_temp_data_file_path**为临时表空间数据文件指定了一个相对路径，如果没有配置，一个自动扩展的12MB的 ibtmp1文件在数据目录被创建。临时表空间在器启动和接收到一个动态空间ID的时候会重新创建，这样就可以避免和已存的空间ID发生冲突。临时表空间不能存在于裸设备上。如果无法创建临时表空间，服务器将不能启动。
临时表空间在正常关闭和异常初始化的时候会自动移除，但发生崩溃的时候不会。这个情况下，管理员要手动移除临时表空间然后重新启动服务器上重新创建临时表空间。
**INFORMATION_SCHEMA.INNODB_TEMP_TABLE_INFO**提供了InnoDB活跃临时表空间的元数据，


# 14.8 InnoDB锁和事务模型
要实现一个大规模，繁忙或高可用的数据库应用，或者提高MySQL的性能，了解InnoDB的锁和事务模型是非常重要的。

这一节讨论了几个关于InnoDB锁和事务模型的主题，你应该熟悉他们才行。


## 14.8.1 InnoDB 锁
本节讨论了InnoDB使用的锁类型。
* (Shared and Exclusive Locks)共享和独占锁  
* (Intention Locks)意图锁（Intention）  
* (Record Locks)记录锁  
* (Gap Locks)间隙锁  
* (Next-Key Locks)Next-Key锁  
* (Insert Intention Locks)插入意图锁  
* (AUTO-INC Locks)AUTO-INC锁

**共享和独占锁**

InnoDB实现了标准的行级锁，有两种类型的锁，`shared(S)`和`exclusive(X)`锁，我们称事务为**T**。
* **S** 允许持有锁的事务**T**读取一行  
* **X** 允许持有锁的事务**T**更新或者删除一行。 

如果事务 T1 在行 r 上持有一个 共享锁 S，那么其他不同事务 T2请求在行 r 上获得一个锁遵守以下规则：  

* T2 可以立刻 S 锁。这样，T1, T2 都会对 r 获得 S 锁  
* T2 不能立刻获得 X 锁  

如果 T1 在行 r 上持有 X 锁，那么 T2 不管是想要一个 S 锁还是  X 锁都无法立即获得。相反，T2 必须等待 T1 释放在 r 上的锁。  

**意图锁**  

InnoDB支持多个粒度的锁，以允许行级锁和表级锁共同工作。在实际场景下，对表级锁被InnoDB称为 意图锁。意图锁是用来表明接下来事务会在表中某一行上请求某个类型的锁（共享或独占）。有两种类型的意图锁。（假设事务 T 在 表 t 上请求一个类型的锁）：  

* **IS**：事务 T 想要在表 t 上的某一行请求 s 锁。  
* **IX**：事务 T 想要在表 t 上的某一行请求 x 锁。  

比如，`SELECT ... LOCK IN SHARE MODE`会设置一个**IS**锁，而`SELECT ... FOR UPDATE`会设置一个 **IX** 锁。

意图锁遵从以下协议：  
* 在事务 T 从表 t 中某一行持有 S 锁前，其必须先在 t 上获得 IS 或 IX。  
* 在事务 T 从表 t 中某一行持有 X 锁前，其必须先在 t 上获得 IX 锁。

可以用以下表来归纳 意图锁与 行级所间的兼容性：  

||X|IX|S|IS|
|---|---|---|---|---|
|X|冲突|冲突|冲突|冲突|
|IX|冲突|兼容|冲突|兼容|
|S|冲突|冲突|兼容|兼容|
|IS|冲突|兼容|兼容|兼容|

如果某个事务请求的锁与已存在的锁兼容，那么就可以获得这把锁，如果与已存锁冲突，那就必须等到已存在的导致冲突的锁释放后才能获得。

因此，意图锁除了全表请求之外（例如 `LOCK TABLES ... WRITE`）并不会阻塞任何事情。IX 和 IS 的主要目的只是用来表明，某些客户端正在锁定表中一行，或者即将锁定表中的某一行。  

一个事务中的意图锁，会在 `SOHW ENGINE INNODB STATUS`和 InnoDB 监控输出中以以下类似的形式显示：  

	TABLE LOCK table `test`.`t.` trx id 10080 lock mode IX


**记录锁**  
记录锁是一个在索引记录上的锁。例如，`SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE`; 将会阻止在 `t.c1`的值是10的行上的 插入，更新或删除操作。  

记录锁总是锁定索引记录，即使表没有定义索引。在这样的情况下，InnoDB会创建一个隐藏的聚簇索引，并用它来进行记录锁。参考**14.11.2.1 Clustered and Secondary Indexes**。  

用`SHOW INNODB ENGINE STATUS`或 InnoDB 监控输出类似以下：

```
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t` 
trx id 10078 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
0: len 4; hex 8000000a; asc     ;;
1: len 6; hex 00000000274f; asc     'O;;
2: len 7; hex b60000019d0110; asc        ;;`iRECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t` 
trx id 10078 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
0: len 4; hex 8000000a; asc     ;;
1: len 6; hex 00000000274f; asc     'O;;
2: len 7; hex b60000019d0110; asc       
```

**间隙锁**  
间隙锁是在索引记录之间的锁定，或者是第一个索引记录之前、最后一个索引记录之后的锁定。比如，`SELECT c1 FROM t BETWEEN 10 and 20 FOR UPDATE`; 会阻止其他事务在列 **t.c1**上插入值 15，不管这一列是不是有这个值，因为在所有在间隙间的值已经被锁定。  

一个间隙里可有只有一个值，或者多个值，也可能是空的。  

间隙锁实在性能和并发性间平衡的结果，只是在某些 隔离级别上使用。  

在使用一个唯一的索引搜索一个唯一的值的的语句中，间隙锁是不需要的。（但这不包括在多列唯一索引上进行进行查询某些列的情况）。比如，如果一个`id`列有一个唯一索引，接下来的语句对id值是100的行使用一个记录锁，同时并不关心其他会话会不会在100之前的间隙中插入数据：  

	SELECT * FROM child WHERE id = 100;

如果`id`没有索引或者是非唯一的索引，那么这个语句就会锁定之前间隙。  

值得注意的是，相冲突的锁可能在一个间隙中被不同的事务所持有。比如，事务 A 在某个间隙上持有一个 共享的间隙锁（gap S-lock），事务 B 在同样的间隙上持有一个独占的间隙锁（gap X-lock）。允许这样的原因是，如果一个记录从索引内清理，被不同事务持有的在这个记录上的锁必须合并。  

间隙锁在InnoDB中说“完全抑制的”，也就是说只是会阻止其他事务插入行到这个间隙。并不会阻止其他事务在同样的间隙上获得间隙锁。因此 **gap X-lock**和**gap S-lock**有同样的影响。  

间隙锁可以明确的禁止。当你把事务隔离级别设置为`READ COMMITTED`，或者设置`innodb_locks_unsafe_for_binlog`系统变量启用就会禁止。在这样的情况下，搜索好索引扫描不会使用间隙锁，只会在外键约束和重复键检查时使用间隙锁。  

使用`READ COMMITTED`和`innodb_locks_unsafe_for_binlog`有其他影响。在MySQL执行完`WHERE`语句后如果没有匹配的行的记录锁就会被释放。对于`UPDATE`语句，InnoDB执行一个“半完整性读”，这样会返回最近提交的版本给MySQL，MySQL以此来判断`UPDATE`中的`WHERE`条件是否匹配。  

**Next-Key锁**   
一个next-key锁是在某个索引记录上的记录锁和这个索引记录前的间隙锁的结合。
当搜索或扫描一个表索引的时候，InnoDB以这样的方式来进行行级锁：在其搜索到的索引记录上设置共享或独占锁。实际上，行级锁就是记录锁。一个在索引记录上的next-key锁也会影响在索引记录之前的间隙。因此，一个Next-key锁就是一个索引记录锁加上在索引记录之前范围的间隙锁。如果一个会话在索引中的一个记录 R 上持有一个共享或独占的锁，另外一个会话就不能在 索引上 R 记录前锁定的间隙中插入新的索引记录。   

假设一个索引包含 10，11，13，20。在这个索引上可能出现的Nexk-key锁有这些（左开右闭：   

```
(negative infinity, 10]
(10,11]
(11,13]
(13,20]
(20,positive infinity)
```

对于最后一种情况，Next-key锁锁定了这个索引中最大值和最大上界（`supremum`一个比索引中任何值大的伪记录）间的间隙。`supremum`不是一个真实的索引记录，因此，实际上，Next-Key锁锁定了在最大值之后的间隙。  

默认情况下，InnoDB工作在`REPEATABLE READ`隔离级别下，同时`innodb_locks_unsafe_for_binlog`系统变量禁止。这样的情况下，InnoDB使用`Next-key`锁来搜索或扫描索引，这样能防止幻行（两次插入同一间隙出现了多出来的结果）。参考**14.8.4 Phantom Rows**

在`SHOW ENGINE INNODB STATS;`下的输出类似如下：  

```
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t` 
trx id 10080 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;
```

 **插入意图锁**  
 一个插入意图锁是一种间隙锁，由`INSERT`操作在插入行之前设置。这个锁，允许不同的事务在同一个间隙的不同位置插入索引记录。假设这里有索引记录，值从4到7。不同的事务试图插入值5，6，每个事务都会在4-7的间隙上获得独占插入意图锁来锁定插入行，但是不会让其他事务等待，因为行并不冲突。  

 接下来的例子展示了一个事务在获取插入记录上的独占锁之前设置一个插入意图锁。这个例子和两个客户端，A B相关。  

 A 建立了一个表，包括两个索引记录（90，102），然后开始一个事务，在ID比100大的索引记录上放一个独占锁。这个独占锁包括一个到102记录前的间隙锁。  
 ```
 ysql> CREATE TABLE child (id int(11) NOT NULL, PRIMARY KEY(id)) ENGINE=InnoDB;
mysql> INSERT INTO child (id) values (90),(102);

mysql> START TRANSACTION;
mysql> SELECT * FROM child WHERE id > 100 FOR UPDATE;
+-----+
| id  |
+-----+
| 102 |
+-----+
 ```

 B开始一个事务，意图在间隙内插入一个记录。在获得独占锁之前，事务先获得一个插入意图锁。  
 ```
mysql> START TRANSACTION;
mysql> INSERT INTO child (id) VALUES (101);
 ```

 插入意图锁的在`SHOW ENGINE INNODB STATUS`中的输出类似下面：  
 ```
 RECORD LOCKS space id 31 page no 3 n bits 72 index `PRIMARY` of table `test`.`child`
trx id 8731 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 80000066; asc    f;;
 1: len 6; hex 000000002215; asc     " ;;
 2: len 7; hex 9000000172011c; asc     r  ;;...
 ```

 **自增锁**  

 一个**AUTO_INC**锁是一个特殊的表级别的锁，这在事务插入具有**AUTO_INCREMENT**列的表时获取。在最简单的情况下，如果A事务正在插入数据到表中，其他事务必须等待A事务获得了连贯的主键值后才能完成插入。  

 `innodb_autoinc_lock_mode`配置选项控制自增锁的算法。这允许你选择如何在可预测的序号和最大的插入操作并发性间进行平衡。  


## 14.8.2 InnoDB Transaction Model
##　14.8.3 Locks Set by Different SQL Statements in InnoDB

一个锁定读，一个`UPDATE`，`DELETE`通常会在处理SQL语句过程中锁定所有扫描过的索引记录。并不关心这些行是不是被一个`WHERE`条件语句所排除。InnoDB不会记住正确的`WHERE`条件，只会记住扫描过的索引范围。通常，锁是`Nexk-key`锁，这会立刻锁住对这条记录前的插入操作。但是呢，`Next-key`锁可以被禁止。对于更多信息，参考**14.18.1 InnoDB Locking**。事务隔离级别也会影响设置什么锁，参考**14.8.2.1 Transaction Isolation Levels**。  

**如果在搜索中对一个二级索引记录上了独占锁，InnoDB会获取对应的聚簇索引记录。**  

对于 `共享锁` 和 `独占锁` 的不同在 **14.8.1 InnoDB Locking**中有介绍。  

如果执行SQL语句时没有合适的索引，MySQL就必须扫描全表，表中所有行都会被锁定，其他用户的插入操作全部需要等待。创建合适的索引以避免不必要的行锁定是非常重要的。  

对于`SELECT ... FOR UPDATE` 或 `SELECT ... LOCK IN SHARE MODE`，被扫描的行需要锁，如果在搜索后的结果集中不会被引用就应该被释放（比如，并不符合在`WHERE`语句中指定的条件）。但是，在某些情况下，行不会被立即解锁，因为在查询期间一个结果行和其来源的关系已经丢失。比如，在一个`UNION`中，扫描过（已锁定）的行可能在验证其是否会在结果集中被引用之前就已经被插入在一个临时表中。在这种情况下，临时表中的行和源表中的行已经没有了关系，所以源表中的行只有在查询结束时被解锁（是不是最后被引用，是对临时表进行条件筛选，而不是源表，这时候）。

InnoDB按以下方式设置锁的类型：  

* `SELECT ... FROM` 是一个一致性读，阅读数据库的一个快照，只在事务隔离级别是**SERIALIZABLE**的时候设置锁。在**SERIALIZABLE**级别下，搜索会在其遇到的索引记录上加上`shared next-key`锁。然而，在一个语句使用一个唯一索引来搜索一个唯一行的时候，只需要一个**记录锁**。  
* `select ... from ... lock in share mode`会在搜索遇到的索引记录上加`shared next-key`锁。然而，在一个语句使用一个唯一索引来搜索一个唯一行的时候，只需要一个**记录锁**。  
* `select ... from ... for update`会在搜索遇到的索引记录上加`exclusive next-key`。然而，在一个语句使用一个唯一索引来搜索一个唯一行的时候，只需要一个**记录锁**。  

对于搜索遇到的索引记录，`select ... from ... for update`会阻塞其他会话执行`select ... from ... lock in share mode`和在特定的事务隔离级别下进行读操作。一致性读会忽略在阅读试图上记录上设置的任何锁。   

* `update ... where ...`会在搜索遇到的任何记录上设置`exclusive next-key`锁，在一个语句使用一个唯一索引来搜索一个唯一行的时候，只需要一个**记录锁**。   
*  当`update`语句更新一个聚簇索引记录的时候，一个隐式的锁会加在受影响的二级索引记录上。`update`操作也会在插入新二级索引记录前做重复检测扫描时，和插入新的二级索引记录过程中，将受影响的二级索引记录上加共享锁。  
*  `delete from ... where ...`会在搜索遇到的任何记录上加上`exclusive next-key`,然而，在一个语句使用一个唯一索引来搜索一个唯一行的时候，只需要一个**记录锁**。  
*  `insert`语句在被插入的行上加`exclusive lock`。这是一个记录锁，而不是`next-key`锁（也就是说，没有gap锁），所以不会阻止其他会话在插入行的前后插入新行。   
在插入行之前，一个叫做插入意图间隙锁的间隙锁被设置。这个锁允许以这样的方式进行插入行：允许不同的事务在同一个间隙内在不同位置插入行而不需要等待。假设这里值为4到7的索引记录。不同的事务方便要插入5，6，在获得对插入行的独占锁之前，都可以同时获得插入意图锁，而不用等待其他事务的完成，因为要插入的行是不冲突的。  
* 如果重复键错误出现，在重复那个索引记录上会设置共享锁。这有可能导致死锁，因为在某个会话已经拥有一个独占锁的情况下，可能有多个会话试图插入相同的行。这种情况可能在其他会话删除行的时候出现。假设 InnoDB 表 t1 有以下结构：  

	create table t1 (i int, primary key (i)) engine = innodb;
	

下面有三个会话按以下的顺序进行操作：  

Session 1:

	start transaction;
	insert into t1 values(1);

Session 2:  

	start transaction;
	insert into t1 values(1);

Session 3:  

	start transaction;
	insert into t1 values(1);

Session 1:

	rollback;

Session 1 在行上请求一个独占锁。Session2, Session3 都会得到一个 `duplicate-key`错误，然后都对这个行上请求一个共享锁。当Session 1回滚，同时释放了独占锁，接下来Session2, Session3所请求的共享锁都会被授权。 在这个时刻，Session2, Session3就发生了死锁：任意一个会话都无法获得独占锁，因为彼此都拥有一个共享锁。    

一个类似的情况可能发生：还有值 1 的行已经存在，然后按以下的方式进行操作：  

Session 1:

	start transaction;
	delete from t1 where i =1;

Session 2:

	start transaction;
	insert into t1 values(1);

Session 3:
	
	start transaction;
	insert into t1 values(1);

Session 1:

	commit;

Session 1的操作需要一个独占锁。Session2, Session3 获得一个 重复键错误 然后同时请求一个共享锁。 当Session 1提交，释放独占锁，Session2, Session3请求的共享锁被授权。 在这个时刻，Session2, Session3发生死锁：由于同时拥有共享锁，所以都等待着获取独占锁。    
* `INSERT ... ON DUPLICATE KEY UPDATE`和一个简单的`INSERT`操作不一样，在一个重复键错误发生的时候，会放置一个独占的`Next-key`锁而不是一个共享锁在将被更新的行上。  
*  `REPLACE`在一个唯一键没有冲突的情况下表现得和`INSERT`一样。否则的话，一个独占的`Next-key`锁会被放在将被替换的行上。  
*  `insert into T select ... from S where ...`会在每一个插入到 T 的行上设置记录锁（没有间隙锁）。如果事务的隔离级别是`READ COMMITTED`，或者`innodb_locks_unsafe_for_binlog`变量已启用且事务隔离级别不是`SERIALIZABLE`，InnoDB在 表 S 上以一致性读进行搜索（没有锁）。否则，InnoDB在表 S 的行上设置一个共享的`Next-key`。InnoDB必须在这种情况下加锁：在恢复数据前滚时，每个SQL语句必须以原来同样的方式执行。  
`create table ... select ...`以一个共享`next-key`锁或一致性读执行`SELECT`语句，和`insert ... select`一样。  
当在`replace into t select ... from s where ...`或`update t ... where col in (select ... from s ...)`中使用`select`时，InnoDB会在表 s 的行上设置共享的`next-key`锁。  
* 在初始化一个表中指定了`AUTO_INCREMENT`的列时，InnoDB在与`AUTO_INCREMENT`列关联的索引尾部设置一个独占锁。I在访问 自增计数器时，InnoDB使用一个叫做`AUTO-INC`的表锁模式，这个锁只持续到语句执行期间，而不是持续到事务结束。在`AUTO-INC`锁持有期间，其他会话不能插入数据到这个表。  **14.8.2 InnoDB Transaction Model**。  
InnoDB获取前一个初始化的`AUTO_INCREMENT`列，不会设置任何锁。  
* 如果在表上设置了外键约束，所有的插入，更新，删除操作需要进行约束条件检测的时候，会其找到来检查约束条件的记录上设置共享的 `record-level`锁。即使是约束检测失败也会设置这些锁。  
* `LOCK TABLES`设置表锁，但这是在MySQL层而不是在之下的InnoDB层设置。如果`innodb_table_lock=1`（默认设置）和`autocommit=0`，InnoDB能识别表锁，MySQL也知道行级锁。  
否则，InnoDB的自动死锁检测在这些表锁被设置的时候无法检测到死锁。因此，在这样的情况下MySQL层不知道行级锁，那么在其他会话在持有一个行级锁的时候可以过得一个表锁。然而，这并不会造成事务完整性的危险，这在**14.8.5.2 死锁检测和回滚**中讨论。同样可以参考**14.11.8 Limits On InnoDB Tables** 
## 14.8.4 Phantom Rows
所谓的`幻行`指的是同一个事务在不同时间的两次查询或者了不同的结果。比如，一个`select`执行两次，第二次的时候获得了第一次没有返回的一行，这后面出来的行就是`幻行`。  
假设，表 child 的 id 列上有一个索引，然后我们想要读取并锁定id>100的所有行，接下来打算更新选中的列上的某些列：  

	select * from child where id > 100 for update;

这个查询从 id > 100 的第一个记录开始扫描索引。我们让这个表包含id的值90，102。如果在扫描过的范围上的记录锁不锁定间隙（这个情况下的间隙是90-102）内的插入，另外一个会话就可以在表内插入一个id=101的行。如果要在同一个事务中执行`select`，你就会看到一个　 id=101的新行（幻行）。如果我们把一系列行视做一个数据条目，这个新的幻行就违背了关于事务的隔离原则：在事务中已读取的数据是不应该改变的。  

为了阻止幻行，InnoDB使用一个结合了索引行锁定和间隙锁的叫做`next-key`的方法。InnoDB以这样的方式实现行级锁，当在搜索或者扫描一个表索引时，会在遇到的索引记录上设置共享或者独占锁。因此，行级锁就是索引记录锁。作为补充，一个索引记录上的`next-key`锁一样影响在索引记录前的间隙。这就是说，`Next-key`锁就是一个索引记录锁加上这个索引记录前的间隙锁。如果一个会话在索引记录 R 上有一个共享或者独占锁，其他会话就不能立刻在R前的这个间隙内插入一个新的索引记录。  

在InnoDB扫描一个索引时，同样可以锁定最后一个记录后的间隙。在前面那个例子中这个情况下出现：为了阻止 id比100大的插入，被InnoDB设置的锁包括了在 id=102后面的间隙。  

可以应用 `next-key` 锁来在应用中实现一个不唯一的检测：如果你以共享模式读取数据，同时看不到你想要插入的行，你可以安全地插入你的行；因为会在阅读期间对插入的行设置`Next-key`锁以阻止其他人插入一个重复的行进来。这就是说，这个`Next-key`锁可以让你“锁定”表中并不存在的东西。  

间隙锁可以在**14.8.1 InnoDB Locking**中讨论的那样禁止。这会导致幻行问题，因为多个会话可以在间隙内插入新行。  

## 14.8.5 Deadlocks in InnoDB
死锁发生在不同的事务都需要获取被对方持有的锁来继续的时候。而因为彼此都需要等待资源可用，所以事务此时也无法释放其获取的锁。  

死锁会在事务通过`UPDATE`或`SELECT ... FOR UPDATE`语句进行锁定多个表的行时发生，以相对的顺序（不懂）。死锁也会在这些语句锁定索引记录范围和间隙的时候发生。死锁例子参考**14.8.5.1 An InnoDB Deadlock Example** 

为了减少死锁的可能，使用事务而不是使用`lock tables`语句；让事务在更新或者插入的时候影响尽量少的数据，并且打开的时间尽量少；当不同事务更新多个表或者很多行的时候，用相同的顺序进行操作（比如`select ... for update`）;在被`select ... for update`语句和`update ... where`语句使用到的列上创建索引。死锁的可能不会被隔离级别影响，因为隔离级别改变了读操作的行为，而死锁经常是由写操作引发。更多关于死锁可能产生的信息，参考**14.8.5.3 How to Minimize and Handle Deadlocks** 

如果死锁发生，InnoDB检测条件，回滚其中一个事务。因此，即使你的引用逻辑是正确的，必须控制一个事务必须重新进行的情况。为了查看上一个出现的死锁的用户事务，使用`show engine innodb status;`命令。如果为了找出经常出现死锁的事务结构或应用错误，设置`innodb_print_all_deadlocks`选项以打印所有的死锁信息到mysql错误日志。关于更多死锁如何自动检测和控制的信息，参考**14.8.5.2 Deadlock Detectiong and Rollback**

## 14.8.5.1  An InnoDB Deadlock Example

接下来的例子显示了当一个锁请求会产生死锁的时候一个错误是怎么样出现的。这个例子和两个客户端相关，A，B。

首先，A 创建一个含有一行的表，然后开始一个事务。在这个事务中，A 通过在共享模式选择这行在这个行上获得 S 锁：

```
ysql> CREATE TABLE t (i INT) ENGINE = InnoDB;
Query OK, 0 rows affected (1.07 sec)

mysql> INSERT INTO t (i) VALUES(1);
Query OK, 1 row affected (0.09 sec)

mysql> START TRANSACTION;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT * FROM t WHERE i = 1 LOCK IN SHARE MODE;
+------+
| i    |
+------+
|    1 |
+------+
```

然后，B 开始一个事务并尝试从表中删除这一行：  

```
mysql> START TRANSACTION;
Query OK, 0 rows affected (0.00 sec)

mysql> DELETE FROM t WHERE i = 1;
```

删除操作需要一个 X 锁。因为 A 拥有一个 S 锁所以这个 X 锁不能获得，因此这个锁会被加入锁请求队列，B会被阻塞。

最后，A也尝试删除这行： 

```
mysql> DELETE FROM t WHERE i = 1;
ERROR 1213 (40001): Deadlock found when trying to get lock;
try restarting transaction
```

死锁就发生了，因为A需要一个X锁来删除这行。然而，这个锁无法被授权因为B也有一个请求X锁在排队等待A来释放S锁。这样被A持有的S锁无法被更新到X锁，因为B对行上X锁的请求排列在前。作为结果，InnoDB会对其中一个锁报错并释放其持有的锁。客户端返回这样的错误：

	ERROR 1213 (40001): Deadlock found when trying to get lock;
	try restarting transaction

这个时候，其他为删除这行而请求的锁请求可以被授权。 

# 14.8.5.2 死锁检测和回滚

InnoDB自动检测事务死锁并且回滚[多个]事务以退出死锁。InnoDB选择小的事务进行回滚，大小是由事务所插入，更新或者删除的行数量来决定。  

当 `innodb_table_locks=1`（默认）或`autocommit=0`的时候，InooDB对表锁是敏感的，而且MySQL层也知道行级锁。否则，InnoDB就不能检测到 MySQL`LOCK TABLES`设置的表锁或非InnoDB引擎设置的锁 引发的死锁。 解决这个情况的办法是设置`innodb_lock_wait_timeout`系统变量。  

当InnoDB完整回滚一个事务的时候，所有这个事务设置的锁都会被释放。然而，只是因为错误而回滚了一个SQL语句的时候，某些被这个语句设置的锁可能会保留。这是因为InnoDB存储表锁以后，其存储的格式无法知道哪个锁是被哪个语句设置的。

如果`select`在一个事务内执行函数，而在函数中的一个语句失败，这个语句会回滚。接下来执行`rollback`的话，整个事务被回滚。

如果InnoDB的监控输出的`LATEST DETECTED DEADLOCK`节包一个信息，`TOO DEEP OR LONG SEARCH IN THE LOCK TABLE EAITS-FOR GARPH, WE WILL ROCK BACK FOLLOWING TRANSACTION`，这表明在等待列表中的事务数已经达到了200的限制。 等待列表中超过200事务被认为是一个死锁，试图检测等待列表的事务被回滚。差不多的情况出现在锁定线程发觉在等待列表中事务拥有的锁超过了1000000个的时候。  

关于如何通过组织数据库操作来避免死锁，参考**14.8.5 Deadlocks in InnoDB**

# 14.8.5.3 How to Minimize and Handle Deadlock


# 14.9 InnoDB Configuration
14.9.1 InnoDB Startup Configuration
14.9.2 InnoDB Buffer Pool Configuration
14.9.3 Configuring the Memory Allocator for InnoDB
14.9.4 Configuring InnoDB Change Buffering
14.9.5 Configuring Thread Concurrency for InnoDB
14.9.6 Configuring the Number of Background InnoDB I/O Threads
14.9.7 Configuring the InnoDB Master Thread I/O Rate
14.9.8 Configuring Spin Lock Polling
14.9.9 Configuring InnoDB Purge Scheduling
14.9.10 Configuring Optimizer Statistics for InnoDB
14.10 InnoDB Tablespaces
14.10.1 Resizing the InnoDB System Tablespace
14.10.2 Changing the Number or Size of InnoDB Redo Log Files
14.10.3 Using Raw Disk Partitions for the System Tablespace
14.10.4 InnoDB File-Per-Table Tablespaces
14.11 InnoDB Tables and Indexes
14.11.1 Creating InnoDB Tables
14.11.2 Role of the .frm File for InnoDB Tables
14.11.3 Physical Row Structure of InnoDB Tables
14.11.4 Moving or Copying InnoDB Tables to Another Machine
14.11.5 Converting Tables from MyISAM to InnoDB
14.11.6 AUTO_INCREMENT Handling in InnoDB
14.11.7 InnoDB and FOREIGN KEY Constraints
14.11.8 Limits on InnoDB Tables
14.11.9 Clustered and Secondary Indexes
14.11.10 Physical Structure of an InnoDB Index
14.12 InnoDB Table Compression
14.12.1 Overview of Table Compression
14.12.2 Enabling Compression for a Table
14.12.3 Tuning Compression for InnoDB Tables
14.12.4 Monitoring Compression at Runtime
14.12.5 How Compression Works for InnoDB Tables
14.12.6 SQL Compression Syntax Warnings and Errors
14.13 InnoDB File-Format Management
14.13.1 Enabling File Formats
14.13.2 Verifying File Format Compatibility
14.13.3 Identifying the File Format in Use
14.13.4 Downgrading the File Format
# 14.14 InnoDB Row Storage and Row Formats
这一节讨论了InnoDB的某些特性是如何被`create table`声明中的`ROW_FORMAT`语句控制的，这些特性包括 **表压缩** 和对于 较长动态长度列值的跨页存储。
## 14.14.1 Overview of InnoDB Row Storage

对行和相关列的存储影响着查询和DML操作的性能。如果多行放在一个磁盘页上，查询和索引查找会更快，innoDB的buffer bool也会需要更少的内存进行缓存, 写出对数字列与短字符列的更新值也会需要更少的I/O。

每个InnoDB表中的数据都被分隔成页。组成每个表的页以树行数据结构进行组织（B-tree索引）。表数据和二级索引都使用这种类型的数据结构。这个代表所有表数据的B-tree索引被称做**聚簇索引**，根据 **主键** 列进行组织。索引数据结构的节点包含了行中所有列的值（聚簇索引）或索引列与主键列（二级索引）。

变长列是一个例外。`BLOB`和`VARCHAR`列因太长而不能放在一个B-tree中，其被放在分别分配的磁盘页上（溢出页）。我们称这些列叫**跨页列**。这些列的值被存储在一个包含溢出页的单向链表中，每个链表有一个或多个溢出页。在某些情况下，这些列值的部分前缀或者全部被存储在B-tree来避免空间浪费和读取多个分离的页。  

`Barracuda`文件格式支持了一个`KEY_BLOCK_SIZE`选项来控制列数据的多少会被存储在聚簇索引中，多少被放在溢出页中。  

接下来的节描述了怎么样来配置InnoDB的行格式来控制变长列值的存储方式。行格式也决定了**表压缩**特性是否支持。  

## 14.14.2 Specifying the Row Format for a Table

用`ROW_FORMAT`语句在`CREATE TABLE`或`ALTER TABLE`声明中来指定行格式。例如：  

	create table t1 (f1 int unsigned) ROW_FORMAT=DYNAMIC engine=INNODB;

InnoDB行格式有`COMPAT，REDUNDANT，DYNAMIC，COMPRESSED`。对于 InnoDB表，COMPACT是默认格式。参考**CREATE TABLE**文档来获得更多关于行格式这个表选项的信息。  

行的物理结构依赖于行格式。参考**14.11.3 Physical Row Structure of InnoDB Tables**

## 14.14.3 DYNAMIC and COMPRESSED Row Formats

这节讨论InnoDB表的**DYNAMIC**和`COMPRESSED`格式。要使用这两种行格式，必须把**innodb_file_format**设置为**Barracuda**，`innodb_file_per_table`也必须启用。（**Barracuda**同样支持**COMPACT**和**REDUNDANT**行格式）

当一个表以`ROW_FORMAT=DYNAMIC`和`ROW_FORMAT=COMPRESSED`进行创建的时候，InnoDB就可以通过跨页存储很长的变长列值（VARCHAR, VARBINARY, BLOB, TEXT列类型），在聚簇索引记录中包含一个20-byte的指针指向溢出页。InnoDB也会将大于等于768bytes的字段编码成变长字段。比如，一个**char(255)**的列，在字符集大于3的时候可以超过768字节（utf8mb4编码）。

列值是否存储跨页依赖于页大小和行的大小。少于或等于40bytes的`TEXT`和`BLOB`列仅以行内方式存储。

`DYNAMIC`行格式会在行能存储在索引节点内的时候提高效率（**COMPACT**和**REDUNDANT**也这样），但是这种格式会将长列的大量数据存储在b-tree节点的问题。`DYNAMIC`来源于这么一个思路，如果一个很长数据的部分值存储在下一页，那还不如把所有的数据都存储在下一页去。以`DYNAMIC`格式存储，更短的行就会更多的留在b-tree节点内，减少了每个列需要的溢出页。

The COMPRESSED row format uses similar internal details for off-page storage as the DYNAMIC row format, with additional storage and performance considerations from the table and index data being compressed and using smaller page sizes. With the COMPRESSED row format, the option KEY_BLOCK_SIZE controls how much column data is stored in the clustered index, and how much is placed on overflow pages. For full details about the COMPRESSED row format, see Section 14.12, “InnoDB Table Compression”.

ROW_FORMAT=DYNAMIC and ROW_FORMAT=COMPRESSED are variations of ROW_FORMAT=COMPACT and therefore handle CHAR storage in the same way as ROW_FORMAT=COMPACT. For more information, see Section 14.11.3, “Physical Row Structure of InnoDB Tables”.


14.14.4 COMPACT and REDUNDANT Row Formats
14.15 InnoDB Disk I/O and File Space Management
14.15.1 InnoDB Disk I/O
14.15.2 File Space Management
14.15.3 InnoDB Checkpoints
14.15.4 Defragmenting a Table
14.15.5 Reclaiming Disk Space with TRUNCATE TABLE
14.16 InnoDB Fast Index Creation
14.16.1 Overview of Fast Index Creation
14.16.2 Examples of Fast Index Creation
14.16.3 Implementation Details of Fast Index Creation
14.16.4 Concurrency Considerations for Fast Index Creation
14.16.5 How Crash Recovery Works with Fast Index Creation
14.16.6 Limitations of Fast Index Creation
14.17 InnoDB Startup Options and System Variables
14.18 InnoDB INFORMATION_SCHEMA Tables
14.18.1 InnoDB INFORMATION_SCHEMA Tables about Compression
14.18.2 InnoDB INFORMATION_SCHEMA Transaction and Locking Information
14.18.3 InnoDB INFORMATION_SCHEMA Buffer Pool Tables
14.19 InnoDB Integration with MySQL Performance Schema
14.19.1 Monitoring InnoDB Mutex Waits Using Performance Schema
14.20 InnoDB Monitors
14.20.1 InnoDB Monitor Types
14.20.2 Enabling InnoDB Monitors
14.20.3 InnoDB Standard Monitor and Lock Monitor Output
14.20.4 InnoDB Tablespace Monitor Output
14.20.5 InnoDB Table Monitor Output
14.21 InnoDB Backup and Recovery
14.21.1 The InnoDB Recovery Process
14.22 InnoDB and MySQL Replication
14.23 InnoDB Troubleshooting
14.23.1 Troubleshooting InnoDB I/O Problems
14.23.2 Forcing InnoDB Recovery
14.23.3 Troubleshooting InnoDB Data Dictionary Operations
14.23.4 InnoDB Error Handling
