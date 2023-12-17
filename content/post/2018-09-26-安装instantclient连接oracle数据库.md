---
title: 安装instantclient连接oracle数据库
categories:
  - 数据库
date: 2018-09-26 13:08:44
updated: 2018-09-26 13:08:44
tags: 
  - Oracle
---
很久没有用。对接上级数据需要用到 Oracle，没法，只能重新捡起来了。 Oracle 官方的文档还是比较完善的，看起来就是比较麻烦。方便下次吧。

# 客户端下载

在页面 [https://www.oracle.com/technetwork/topics/linuxx86-64soft-092277.html](https://www.oracle.com/technetwork/topics/linuxx86-64soft-092277.html) 下载好对于的 instant 与 sqlplus 版本。我下载的是 12.2 版本。

我下载的是当前用户的 Home 目录下，也就是 `~`。把他们解压到一起：

```shell
cd
unzip instantclient-basic-linux.x64-12.2.0.1.0.zip
unzip instantclient-sqlplus-linux.x64-12.2.0.1.0.zip
```

下载完毕后可以看一下都有些什么文件：

```shell
ls -1 instantclient_12_2
adrci
BASIC_README
genezi
glogin.sql
libclntshcore.so.12.1
libclntsh.so
libclntsh.so.12.1
libipc1.so
libmql1.so
libnnz12.so
libocci.so.12.1
libociei.so
libocijdbc12.so
libons.so
liboramysql12.so
libsqlplusic.so
libsqlplus.so
ojdbc8.jar
sqlplus
SQLPLUS_README
uidrvci
xstreams.jar
```
安装官方文档上的说明还有一些后续步骤需要做： [https://www.oracle.com/technetwork/database/features/instant-client/sqlplus-cloud-3080557.html](https://www.oracle.com/technetwork/database/features/instant-client/sqlplus-cloud-3080557.html)

**动态连接库**

```shell
cd instantclient_12_2
ln -s libclntsh.so.12.1 libclntsh.so
```
建立这么一个软连接后，需要在动态库寻找路径里面加入这个。有两种方法可以做到。

```shell
export LD_LIBRARY_PATH=~/instantclient_12_2:$LD_LIBRARY_PATH
```

或者在配置文件内指定：

```shell
echo /home/myuser/instantclient_12_2 > /etc/ld.so.conf.d/oic.conf
ldconfig
```

相关环境变量设置：

```shell
export ORACLE_HOME=~/instantclient_12_2
export TNS_ADMIN=~/instantclient_12_2
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$ORACLE_HOME
export PATH=$PATH:$ORACLE_HOME
export NLS_LANG="AMERICAN_AMERICA.AL32UTF8"  #这个是为了防止乱码
```

# 连接
我们可以采用  tnsname.ora 来连接，或者直接用命令行连接：

```
cd instantclient_12_2
touch tnsname.ora

```

**命令行连接**

```shell
sqlplus user/password@//172.230.1.11:1521/topicis
```

# 数据导出为csv
因为想要将数据导出，然后装到 MySQL 去，所以选择了以 csv 的形式，可更方便一些。关于数据的导出， Oracle 很多都是用的 pl/sql 或者现在新出品的 sql developer，但是我需要的是定时任务自动执行这样，所以只能使用 sqlplus 来进行导出来。

主要就是利用 sqlplus 的 spool 命令来把显示内容转存到文件中。我们可以用一个简单的示例来展示：

```sql
sqlplus user/password@//172.230.1.11:1521/topicis << EOF
spool test.csv
select sysdate from dual;
spool off;
EOF
```

事实上我们可以把想要执行的命令放到一个 sql 文件中，然后以 @filename 的形式来调用。比如上面的语句我们就可以放在一个文件 test.sql 中：
```sql
spool test.csv
select sysdate from dual;
spool off;
```

```sql
sqlplus user/password@//172.230.1.11:1521/topicis << EOF
@test
EOF
```

