---
title: CentOS安装Python3.7
categories:
  - Python
date: 2020-02-14 21:45:24
updated: 2020-02-14 21:45:24
tags: 
  - Python
  - CentOS
---
CentOS 默认使用的是 python2，特别是 yum 使用的也是 python2，如果我们贸然的安装了 python3 的话会造成无法使用 yum ，这个是有个教训的。所以现在就来特别整合看一下如何安装 python3。最好最麻烦的也就是使用源码安装了。

<!--more-->

# 依赖安装

```sh
yum install gcc openssl-devel bzip2-devel libffi-devel
```

# 源码下载

```sh
cd /usr/src
wget https://www.python.org/ftp/python/3.10.0/Python-3.10.0.tgz
tar xzf Python-3.10.0.tgz

wget https://www.openssl.org/source/openssl-1.1.1l.tar.gz
tar xzf openssl-1.1.1l

```

# 编译安装

```sh
cd openssl-1.1.1l && ./config prefix=/usr/local/openssl1.1.1l && make && make install_sw && cd ..

cd Python-3.10.0
./configure --enable-optimizations --with-openssl=/usr/local/openssl1.1.1l
make altinstall -j 8
```
`altinstall` 不会替换默认的 **/usr/bin/python**

# 版本检查

```sh
python3.10 -V
```
# Sqlite3

yum 安装的 sqlite3 是 3.7 版本的  django 用不了，所以要安装新版本的。

```sh
cd /usr/src/
wget https://sqlite.org/2020/sqlite-autoconf-3310100.tar.gz
tar xzvf sqlite-autoconf-3310100.tar.gz
cd sqlite-autoconf-3310100
./configure
make && make install
```

然后重新编译 python3.10.0

# PIP 加速设置

```sh
mkdir -p ~/.pip
touch ~/.pip/pip.conf

tee ~/.pip/pip.conf <<-'EOF'
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
EOF

```
