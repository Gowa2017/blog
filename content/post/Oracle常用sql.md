---
title: Oracle常用sql
toc: true
date: 2016-11-29 10:24:33
tags: [Oracle, 数据库]
categories: 
  - 数据库
---
主要是工作记录需要应用到的时候进行了一下记录，包括会话、session、锁等的查询。
<!--more-->
# 查询应用、连接数

	SELECT  B.PROGRAM , COUNT(1) 
	FROM V$PROCESS A, V$SESSION B 
	WHERE A.ADDR = B.PADDR 
	AND  B.USERNAME IS NOT NULL
	GROUP BY B.PROGRAM;
# 历史最大会话数：

	SELECT SESSIONS_MAX,SESSIONS_WARNING,SESSIONS_CURRENT,SESSIONS_HIGHWATER FROM v$license;
# 查看session参数

	SELECT NAME, TYPE, VALUE FROM V$PARAMETER WHERE NAME LIKE 'session%';
# 查询被锁的表并释放session

	SELECT A.OWNER
	  ,A.OBJECT_NAME
	  ,B.XIDUSN
	  ,B.XIDSLOT
	  ,B.XIDSQN
	  ,B.SESSION_ID
	  ,B.ORACLE_USERNAME
	  ,B.OS_USER_NAME
	  ,B.PROCESS
	  ,B.LOCKED_MODE
	  ,C.MACHINE
	  ,C.STATUS
	  ,C.SERVER
	  ,C.SID
	  ,C.SERIAL#
	  ,C.PROGRAM
	FROM ALL_OBJECTS A,V$LOCKED_OBJECT B,SYS.GV_$SESSION C
	WHERE  A.OBJECT_ID = B.OBJECT_ID  AND B.PROCESS = C.PROCESS  ORDER BY 1,2;
# 查看系统IO较大的session

	SELECT se.sid
	      ,se.serial#
	      ,pr.spid
	      ,se.username
	      ,se.status
	      ,se.terminal
	      ,se.program
	      ,se.module
	      ,se.sql_address
	      ,st.event
	      ,st.p1text
	      ,si.physical_reads
	      ,si.block_changes
	FROM v$session se,v$session_wait st,v$sess_io si,v$process pr
	WHERE st.sid=se.sid  AND st.sid=si.sid 
	  AND se.paddr=pr.ADDR AND se.sid>6
	  AND st.wait_time=0 AND st.event NOT LIKE '%SQL%' 
	  ORDER BY physical_reads DESC;

# 查询消耗cpu较多的session

	select a.sid
	      ,spid
	      ,status
	      ,substr(a.program,1,40) prog
	      ,a.terminal
	      ,osuser
	      ,value/60/100 value
	from v$session a,v$process b,v$sesstat c
	where c.statistic#=12 and c.sid=a.sid and a.paddr=b.addr
	   order by value desc

