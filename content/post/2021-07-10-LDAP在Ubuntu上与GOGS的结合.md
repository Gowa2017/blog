---
title: LDAP在Ubuntu上与GOGS的结合
categories:
  - Linux/Unix
date: 2021-07-10 13:21:39
updated: 2021-07-10 13:21:39
tags: 
  - Linux/Unix
  - LDAP
  - Gogs
---
最新公司需要做统一的文档管理、代码管理。不想要放在外部的网站上，因此呢，就需要自己建立一个。我的思路是使用  LDAP 来进行统一的账号管理，然后将版本管理库、文档权限这些都用 LDAP 来简单的进行管理就行了。

<!--more-->

# GOGS

GOGS 的安装，不用多言，查看官方文档就行了 [https://gogs.io/docs/installation](https://gogs.io/docs/installation)，最简单的就是下载 二进制程序，然后直接执行 `./gogs web`，然后进入对应的页面，进行配置就好了。

我为了偷懒，使用了 SQLITE3 数据库，就不用装什么其他的了。

# LDAP

这个我们用 openldap 来部署了。非常的简单。

```sh
sudo apt-get install slapd ldap-utils
sudo dpkg-reconfigure slapd
```

按照下一步下步进行安装就行了。

## ldapscript

我们可以使用 ubuntu 提供的脚本来简化我们对 ldap 的管理。

```sh
sudo apt install ldapscripts
```

修改 `etc/ldapscripts/ldapscripts.conf` 文件：

```
SERVER=ldap://ldap01.example.com
LDAPBINOPTS="-ZZ"
BINDDN='cn=admin,dc=example,dc=com'
BINDPWDFILE="/etc/ldapscripts/ldapscripts.passwd"
SUFFIX='dc=example,dc=com'
GSUFFIX='ou=Groups'
USUFFIX='ou=People'
MSUFFIX='ou=Computers'
```
然后将我们的明文密码写入到 `/etc/ldapscripts/ldapscripts.passwd"` 里面，不使用加密的话，把 `-ZZ` 替换成空就行了。

因为这个脚本会将用户和组添加到预设的 OU 中，所以我们还需要自己来添加好几个OU。采用自己编写文件的方式：

```
dn: ou=Users,dc=gowa,dc=club
objectClass: organizationalUnit
ou: Users

dn: ou=Groups,dc=gowa,dc=club
objectClass: organizationalUnit
ou: Groups
```

执行 ：

```
ldapadd -x -D cn=admin,dc=gowa,dc=club -W -f a.ldif
```

这样我们可以很愉快的使用  `ldapadduser user group` 的形式来管理用户了。

# 与 GOGS 接入

认证源管理设置：

```
认证类型：LDAP（Via BindDN）
认证名称：自己写
认证议： UnCrypted
主机地址： 域名或者IP
主机端口：默认 389
绑定DN： cn=admin,dc=gowa,dc=club
绑定密码： admin 的密码
用户搜索基准：(&(ou=Users,dc=gowa,dc=club))
用户过滤规则：(&(objectClass=posixAccount)(uid=%s)
邮箱属性： main
```

这样搞了后就可以直接用 LDAP 的账号登录了。

# NGINX 的接入

这个需要手动来编译才行。

```sh
cd /usr/local/src 
curl -O http://nginx.org/download/nginx-1.20.1.tar.gz
tar xzvf nginx-1.20.1.tar.gz
git clone https://gitclone.com/github.com/kvspb/nginx-auth-ldap.git
cd nginx-1.20.1
./configure --add-module=../nginx-auth-ldap
make && make install
```

然后到  /usr/local/nginx/conf 里面进行配置：

```
http{
    ...
        ldap_server openldap {

        url ldap://localhost/dc=gowa,dc=club?uid?sub?(&(objectClass=account));

        binddn "cn=admin,dc=gowa,dc=club";

        binddn_passwd "wouinibaba";

        group_attribute memberuid;

        group_attribute_is_dn on;

        require valid_user;

      }
      server {
        location / {
            ...
            auth_ldap "Forbidden";
            auth_ldap_servers openldap;

        }
      }
}
```
