---
title: Oracle数据字典检查
date: 2016-10-31 15:00:27
tags: [Oracle, 数据库]
categories: 
  - 数据库
toc: true
---
作为一个Oracle菜鸟想要管理数据库，怎么能不看一下官方的文档呢，日常所用的检查也能了解得差不多了。表空间、数据文件、控制文件段等等都可以得到很好的解答。
<!--more-->
那么下面，我们开始了。
# 查看数据库及链接库
	SELECT * from v$database;
	SELECT * from dba_db_links;
# 查看控制文件   
	SELECT * FROM V$CONTROLFILE;
# 查看数据在线重做日志文件
	SELECT * FROM V$LOG;
	SELECT * FROM V$LOGFILE;
# 查看回滚表空间
	SELECT OWNER,SUM(BYTES)/1024/1024 AS "I_SIZE/MB" 
	FROM DBA_UNDO_EXTENTS GROUP BY OWNER;
# 查看数据文件与表空间关联   
	SELECT * FROM DBA_DATA_FILES;
# 查询某一表空间的数据文件
	SELECT file_name,bytes/1024/1024 as "size",online_status 
	FROM dba_data_files WHERE tablespace_name='OTHER_DATA';
# 查看表空间大小
	SELECT TABLESPACE_NAME,SUM(BYTES)/1024/1024 AS "TOTAL/MB" 
	FROM DBA_DATA_FILES GROUP BY TABLESPACE_NAME 
	ORDER BY TABLESPACE_NAME;
# 查看空闲表空间
	SELECT TABLESPACE_NAME,SUM(BYTES)/1024/1024 AS "FREE / MB" 
	FROM DBA_FREE_SPACE GROUP BY TABLESPACE_NAME;
# 查看某一表空间内使用最多的表
	SELECT SEGMENT_NAME,SUM(BYTES)/1024/1024 AS USED 
	FROM DBA_SEGMENTS T WHERE TABLESPACE_NAME='OTHER_INDEX' 
	GROUP BY SEGMENT_NAME ORDER BY USED DESC;
# 查看所有表与表空间关联
	SELECT * FROM USER_TABLES;
# 查看表空间使用情况
	SELECT tbs tablespace_name, sum(totalM), sum(usedM), sum(remainedM), 
		round(sum(usedM)/sum(totalM)*100,2) as "use%", round(sum(remainedM)/sum(totalM)*100,2) as "free%" 
	FROM(SELECT b.file_id ID,  b.tablespace_name tbs,  b.file_name name,  b.bytes/1024/1024 totalM,  
		(b.bytes-sum(nvl(a.bytes,0)))/1024/1024 usedM,  sum(nvl(a.bytes,0)/1024/1024) remainedM,  
		sum(nvl(a.bytes,0)/(b.bytes)*100),  (100 - (sum(nvl(a.bytes,0))/(b.bytes)*100))  
		FROM dba_free_space a,dba_data_files b WHERE a.file_id = b.file_id  
		GROUP BY b.tablespace_name,b.file_name,b.file_id,b.bytes 
		ORDER BY b.tablespace_name)
	GROUP BY tbs;
# 为表空间增加一个数据文件
	ALTER tablespace_name ADD DATAFILE 'data2.ora';
# 表空间数据文件设置为自增
	ALTER DATABASE DATAFILE 'data3.ora' AUTOEXTEND ON NEXT 50M MAXSIZE 1000M;
# 新建一个表空间
	CREATE tablespace_name DATAFILE 'data4.ora' SIZE 2G;
