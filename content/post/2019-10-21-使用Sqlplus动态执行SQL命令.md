---
title: 使用Sqlplus动态执行SQL命令
categories:
  - 数据库
date: 2019-10-21 11:00:04
updated: 2019-10-21 11:00:04
tags: 
  - Oracle
  - Sqlplus
---
之前都是要执行的命令写在 SQL 文件中，然后用 @filename 的形式来进行调用。但是这样的话，里面很多需要进行变更的参数就无法做到动态替换，总不可能为每个命令都写一个文件撒，所以就研究了一下 SqlPlus 更多的用法

<!--more-->

# 命令格式

```sh
sqlplus [ [<option>] [{logon | /nolog}] [<start>] ]
```

分三个部分：

- options 指定各种选项 我们主要使用  **-S** 来实现静默执行。
- logon 指定登录的方式。`{<username>[/<password>][@<connect_identifier>] | / }
	      [AS {SYSDBA | SYSOPER | SYSASM | SYSBACKUP | SYSDG | SYSKM | SYSRAC}] [EDITION=value]` 我们采用账号密码的形式登录，然后也不作为管理员登录那么就简单的这样用就行了:`sqlplus myusername/mypassword@//IP:port/INSTANCE`
- start `@<URL>|<filename>[.<ext>] [<parameter> ...]` 指定从一个网页服务器 URL 或者本地文件系统，配合对应的 parameter 变量来进行执行。

> 当 Sql启动后，在 CONNECT 命令后，全局配置文件
(e.g. $ORACLE_HOME/sqlplus/admin/glogin.sql) 和用户配置文件(e.g. login.sql in the working directory) 都会被运行。这些文件可能会包含相应的命令。
所以我们可以将常用的一些行列显示的写到当前工作目录的 login.sql 文件中。
那么我们主要关注 start 部分的操作方式就行了。

# SQL 文件中变量的指定

我们在 SQL 文件中可以用 `&1 &2` 的形式来指定变量，对应我们传递过去的参数。

如 文件中 **test.sql** 中我们写入：

```sql
select &1 from dual;
```

我们也命令执行的时候：

```sql
sqlplus -S username/password//ip:1521/port @test "520102"

old   1: select &1 from dual
new   1: select 520102 from dual

    520102
----------
    520102
```
输入就已经被替换了。我们来看更复杂一点的我们用来导出主体的例子：

```sql
set colsep ,
set heading on
set headsep on
set linesize 5000
set pagesize 0
set trimspool on
set feedback off
set echo off
set term off

alter session set nls_date_format='yyyy-mm-dd hh24:mi:ss';

spool csv/REG.csv

SELECT
a.ID || '^~^' || 
to_char(a.TIMESTAMP,'YYYY-MM-DD HH:MI:SS') || '^~^' || 
a.NAMEPREAPPRID || '^~^' || 
a.ENTNAME || '^~^' || 
a.ENTTRA || '^~^' || 
a.GRPSHFORM || '^~^' || 
a.OPLOCDISTRICT || '^~^' || 
a.INDUSTRYPHY || '^~^' || 
a.INDUSTRYCO || '^~^' || 
a.LEREP || '^~^' || 
a.REGCAP || '^~^' || 
a.REGCAPCUR || '^~^' || 
a.RECCAP || '^~^' || 
a.FORRECCAP || '^~^' || 
a.FORREGCAP || '^~^' || 
a.CONGRO || '^~^' || 
a.DOM || '^~^' || 
a.TEL || '^~^' || 
a.POSTALCODE || '^~^' || 
a.EMAIL || '^~^' || 
a.ABUITEMCO || '^~^' || 
a.OPSCOPE || '^~^' || 
a.PTBUSSCOPE || '^~^' || 
a.ENTTYPE || '^~^' || 
a.ENTTYPEITEM || '^~^' || 
a.ENTTYPEMINU || '^~^' || 
to_char(a.OPFROM,'YYYY-MM-DD HH:MI:SS') || '^~^' || 
to_char(a.OPTO,'YYYY-MM-DD HH:MI:SS') || '^~^' || 
a.REGNO || '^~^' || 
a.OLDREGNO || '^~^' || 
a.FORREGNO || '^~^' || 
a.SUPERVPER || '^~^' || 
a.SUPERORGID || '^~^' || 
to_char(a.ESTDATE,'YYYY-MM-DD HH:MI:SS') || '^~^' || 
to_char(a.APPRDATE,'YYYY-MM-DD HH:MI:SS') || '^~^' || 
a.PERID || '^~^' || 
a.ACCOPIN || '^~^' || 
a.REMARK || '^~^' || 
a.STATE || '^~^' || 
a.ORGID || '^~^' || 
a.JOBID || '^~^' || 
a.ADBUSIGN || '^~^' || 
a.TOWNSIGN || '^~^' || 
a.REGTYPE || '^~^' || 
a.PRIORGID || '^~^' || 
a.SUPERPRIORGID || '^~^' || 
a.APPRORGID || '^~^' || 
a.ENTTYPEPRO || '^~^' || 
a.OPTYPE || '^~^' || 
a.EMPNUM || '^~^' || 
a.COMPFORM || '^~^' || 
a.SUPDISTRICT || '^~^' || 
a.VENIND || '^~^' || 
a.PARNUM || '^~^' || 
a.EXENUM || '^~^' || 
a.OPFORM || '^~^' || 
a.INSFORM || '^~^' || 
a.HYPOTAXIS || '^~^' || 
a.FORCAPINDCODE || '^~^' || 
a.MIDPREINDCODE || '^~^' || 
a.PROTYPE || '^~^' || 
a.IMPDATESIGN || '^~^' || 
a.OPLOC || '^~^' || 
a.COPYNUM || '^~^' || 
a.ENTTYPEGB || '^~^' || 
a.COMPFORMGB || '^~^' || 
a.HOTINDFOCUS || '^~^' || 
a.PARFORM || '^~^' || 
a.INDUSTRYPHYGB || '^~^' || 
a.INDUSTRYCOGB || '^~^' || 
a.APPPERID || '^~^' || 
a.UNISCID || '$'
FROM
topicis.REG_MARPRIPINFO a 
WHERE 
    a.timestamp > trunc(SYSDATE - 1,'DD')
--    trunc(a.timestamp,'DD') = '2019-03-25'
    AND ( a.oplocdistrict = &1
          OR a.orgid IN (
        SELECT
            b.id
        FROM
            topicis.a_organ b
        WHERE
            b.id = '152010200000000'
            OR b.parent = '152010200000000'
    )) and rownum < 10
ORDER BY a.id;
spool off
```
依然如先前那样执行，就OK了。

但是发现一个问题，输出文件中会包含：

```
old  79:     AND ( a.oplocdistrict = &1
new  79:     AND ( a.oplocdistrict = 520102
```

变量替换也给输出了，这个需要解决。

在[StackOverFlow 上找到了答案](https://stackoverflow.com/questions/5340716/suppress-output-of-variables-substitution-in-sqlplus)  


```sql
SET VERIFY OFF
```

就OK 

# login.sql

我们可以把我们公用的配置全部都放在这个文件内，然后放到当前执行的目录。但是从 12.2 and in 11.2.0.4 db version after a 2017 PSU patch,  这种方式已经不能正确的工作了。最后是在 [这里找到了解决方法](http://dbaparadise.com/2017/08/how-to-make-login-sql-work-again-in-12-2-and-11-2-0-4/)

> SQLPlus will only look in $ORACLE_PATH environment variable on Unix, and %SQLPATH% on Windows for the login.sql

只会查找设置的环境变量中的文件了不会查找当前路径下的文件了。

```sql
set colsep ,
set heading on
set headsep on
set linesize 5000
set pagesize 0
set trimspool on
set feedback off
set verify off
set echo off
set term off

alter session set nls_date_format='yyyy-mm-dd hh24:mi:ss';
```

我们需要设置好 ORACLE_PATH 变量的路径，并将其 export ，否则的话，我们在 Shell 脚本内设置，但其子进程是不会继承 这个环境变量的。
