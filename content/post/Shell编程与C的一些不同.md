---
title: Shell编程与C的一些不同
date: 2016-11-13 09:37:17
tags: [Linux, Shell]
toc: true
categories: [Linux/Unix]
---

事情是这样的，前几天整理一下业务脚本的时候，发现有几处写的地方让我莫名其妙，后面经过仔细的了解才明白，确实有点反常。
<!--more-->
# 开始不明白的例子
观察以下代码，按照逻辑，`if condition; then ...` 才会执行，同学们来看看这会出现什么。
```sh
CheckDb()
{
ret=`sqlplus -s absadmin/imsadmin32@LCIMS80 << !
set pagesize 0
set feedback off
select 'ok' from dual;
!`
	ret=`echo $ret`
        if [ "${ret}" != "ok" ]
        then
                echo "`date +'[%Y-%m-%d %H:%M:%S]'` ${ret}"
		echo 1
                return 1
        fi
	echo 0
        return 0
}
for times in 1 2 3
do
	if CheckDb
        then
                auth0=`ps -ef | grep 'AB_AUTH 0' | grep -v monitor | grep -v grep | wc -l`

                if [ ${auth0} -gt 0 ]
                then
                        echo "`date +'[%Y-%m-%d %H:%M:%S]'`  resume from no DB !"
                        SendMsg "resume+from+no+DB"
                        sh ${AAA_HOME}/bin/restartall
                fi
                break
        fi

	if [ $times -eq 3 ]
        then
                AuthNoDB
                break
        fi

        SendMsg "login+DATABASE+fail+$times+times"
        sleep 3
done
i
```

事实上，shell的if中的condition有三种模式。
`if command; then ...; fi` 这种情况下`command(可以是函数)`执行成功，返回0,那么执行then后面的语句，而不是进行T|F判断;
`if [ condition ]; then ...; fi` 这种情况下`condition`为真才会执行then。

	if [ 1 -gt 2 ]; then echo true; else echo false; fi
`if test condition; then ...; fi` 这种情况下`condition`为真才会执行then。   

	if test 1 -gt 2 ; then echo true; else echo false; fi
因为在shell中 `test 与 []` 是完全一样的。
