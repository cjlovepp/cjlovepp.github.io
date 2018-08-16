---
title: Linux环境安装Postgresql
date: 2018-08-16 20:00:51
tags:
    - postgresql
categories:
    - 数据库
---

## 前言
- Linux版本：CentOS
- PostgresSQL版本：9.5.6

<!--more-->

## 安装

### 下载安装软件包
可直接去官网下载，下载后直接找个目录放进去就行，我这边的目录为'/usr/lib/pg/' **注意：该目录不是安装目录，所以安装完成后即可删除**

### 解压
```console
tar -zxvf postgresql-9.5.6.tar.gz
```

### 编译安装
```console
./configure

make

make install
```
- ./configure是检查当前环境能否安装PG，同时也可以使用 `./configure --help`查看参数命令
> 这里注意下`--prefix=PREFIX`该参数用于指定pg的安装目录，默认是`/usr/local/pgsql`这里我直接使用默认值。

> ./configure检测可能会提示系统未安装gcc等lib库，根据提示安装即可

### 创建postgres用户
posgresql默认使用postgres用户
```console
adduser postgres

mkdir /usr/local/pgsql/data

chown postgres /usr/local/pgsql/data
```

### 配置数据库
配置远程访问需要修改两个配置文件`postgresql.conf`和`pg_hba.conf`
#### 修改监听地址
```conf
#listen_addresses=’localhost’
#将上面这行改成如下
listen_addresses=’*’
```

#### 添加IP授权
```conf
# 这是在/pgsql/data/pg_hba.conf文件里加
# IPv4 myhost connections:
host    all         all         0.0.0.0/0          trust
```
这里设置了对所有IP开发，可修改`trust`为`password`为远程登录设置密码，重启后生效

#### 启动/重启
```console
bin/pg_ctl start -D /mount/pgsql_data/ -l /mount/pgsql_log  //重启restart
```

### 设置远程登陆账户和密码
服务器本地登陆postgresql数据库（默认是不需要密码的）
```console
[postgres@localhost pgsql]$ pwd
/usr/local/pgsql
[postgres@localhost pgsql]$ bin/psql

psql.bin (9.5.9)

Type "help" for help.
```

- 创建角色，并且给角色设置密码:
```console
postgres=# create user testwjw with password 'Zykj@5&^%996';

CREATE ROLE
```
- 修改数据库用户和密码：
```console
postgres=# alter user testwjw with password '558996';

ALTER ROLE
```

- 指定字符集创建数据库testdb1，并且授权给testwjw
```console
postgres=# create database testdb1 with encoding='utf8' owner=testwjw;

CREATE DATABASE
```
- 授权：
```console
postgres=# grant all privileges on database testdb1 to testwjw; 

GRANT
```

#### 修改postgresql.conf文件中的端口和监听主机:

postsql默认安装后是监听本机127.0.0.1 默认端口是5432,是不能够远程登陆的，所以要修改监听主机地址postgresql数据库的配置文件是:postgresql.conf,所在位置是：postgresql初始化时所指定的data数据目录下：具体可参照上文的配置

#### 重启postgresql服务生效：
