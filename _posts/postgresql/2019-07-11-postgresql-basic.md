---
title: "postgresql 安装与常用操作"
tags: [postgresql]
---

# 一、安装

首先，安装PostgreSQL客户端。

```sh
sudo apt-get install postgresql-client
```

然后，安装PostgreSQL服务器。

```sh
sudo apt-get install postgresql
```

正常情况下，安装完成后，PostgreSQL服务器会自动在本机的5432端口开启。

如果还想安装图形管理界面，可以运行下面命令，但是本文不涉及这方面内容。

```sh
sudo apt-get install pgadmin3
```

# 二、添加新用户和新数据库

初次安装后，默认生成一个名为 `postgres` 的数据库和一个名为 `postgres` 的数据库用户。这里需要注意的是，同时还生成了一个名为 `postgres` 的Linux系统用户。

下面，我们使用 `postgres` 用户，来生成其他用户和新数据库。好几种方法可以达到这个目的，这里介绍两种。

## PostgreSQL控制台

首先，新建一个Linux新用户，可以取你想要的名字，这里为dbuser。

```sh
sudo adduser dbuser
```

然后，切换到 `postgres` 用户。

```sh
sudo su - postgres
```

下一步，使用 `psql` 命令登录PostgreSQL控制台。

```sh
psql
```

这时相当于系统用户 `postgres` 以同名数据库用户的身份，登录数据库，这是不用输入密码的。如果一切正常，系统提示符会变为"`postgres=#`"，表示这时已经进入了数据库控制台。以下的命令都在控制台内完成。

第一件事是使用 `\password` 命令，为 `postgres` 用户设置一个密码。

```sh
\password postgres
```

第二件事是创建数据库用户 `dbuser`（刚才创建的是Linux系统用户），并设置密码。

```sql
CREATE USER dbuser WITH PASSWORD 'password';
# 删除用户
drop user dbuser;
```

第三件事是创建用户数据库，这里为 `exampledb` ，并指定所有者为 `dbuser` 。

```sql
CREATE DATABASE exampledb OWNER dbuser;
```

第四件事是将 `exampledb` 数据库的所有权限都赋予 `dbuser` ，否则 `dbuser` 只能登录控制台，没有任何数据库操作权限。

```sql
GRANT ALL PRIVILEGES ON DATABASE exampledb to dbuser;
```

最后，使用 `\q` 命令退出控制台（也可以直接按`ctrl+D`）。

```sql
\q
```

## shell命令行

添加新用户和新数据库，除了在PostgreSQL控制台内，还可以在shell命令行下完成。这是因为PostgreSQL提供了命令行程序 `createuser` 和 `createdb` 。还是以新建用户 `dbuser` 和数据库 `exampledb` 为例。

首先，创建数据库用户 `dbuser` ，并指定其为超级用户。

```sh
sudo -u postgres createuser --superuser dbuser
```

然后，登录数据库控制台，设置 `dbuser` 用户的密码，完成后退出控制台。

```sql
sudo -u postgres psql

\password dbuser

\q
```

接着，在shell命令行下，创建数据库 `exampledb` ，并指定所有者为 `dbuser` 。

```sh
sudo -u postgres createdb -O dbuser exampledb
```

# 三、登录数据库

添加新用户和新数据库以后，就要以新用户的名义登录数据库，这时使用的是 `psql` 命令。

```sh
psql -U dbuser -d exampledb -h 127.0.0.1 -p 5432
```

上面命令的参数含义如下：`-U` 指定用户，`-d` 指定数据库，`-h` 指定服务器，`-p` 指定端口。

输入上面命令以后，系统会提示输入 `dbuser` 用户的密码。输入正确，就可以登录控制台了。

psql命令存在简写形式。如果当前Linux系统用户，同时也是PostgreSQL用户，则可以省略用户名（ `-U` 参数的部分）。举例来说，前例中切换用户后 Linux系统用户名为 `postgres` ，且PostgreSQL数据库存在同名用户 `postgres` ，则我以 `postgres` 身份登录Linux系统后，可以直接使用下面的命令登录数据库，且不需要密码。

```sh
psql exampledb
```

此时，如果PostgreSQL内部还存在与当前系统用户同名的数据库 `postgres`，则连数据库名都可以省略。比如，假定存在一个叫做 `postgres` 的数据库，则直接键入psql就可以登录该数据库。

```sh
psql
```

另外，如果要恢复外部数据，可以使用下面的命令。

```sh
psql exampledb < exampledb.sql
```

# 四、控制台命令

除了前面已经用到的 `\password` 命令（设置密码）和 `\q` 命令（退出）以外，控制台还提供一系列其他命令。

```sh
\?：# 查看psql命令列表。
\c [database_name]：# 切换到其他数据库
\l：# 列出所有数据库。
\d：# 列出当前数据库的所有表格。
\d [table_name]：#schema.table 查看表的结构 
\du：列出所有用户。
\e：打开文本编辑器。
\conninfo：列出当前数据库和连接的信息。
\h：# 查看所有的sql关键字 或 查看SQL命令的解释，比如\h select。
\q      #退出pg命令行
\x      #横纵显示切换
\dT+    #显示扩展类型相关属性及描述
\dx     #显示已安装的扩展插件
\d 数据库 #得到所有表的名字
\d 表名 # 得到表结构
\timing #显示执行时间
set search to schema    #切换schema
explain sql             #解释或分析sql执行过程

```

# 五、数据库操作

基本的数据库操作，就是使用一般的SQL语言。
**PS:** 语句执行的时候应该以 `;` 结束才表示语句完成。否则会被默认为语句还没有写完，不会执行。
```sql
# 列出数据库名
\l
# 或 
SELECT datname FROM pg_database;

# 切换数据库
\c 数据库名

# 删除库
\c hello2;
DROP DATABASE IF EXISTS hello;

# 列出当前数据库所有表
\dt
# 列出表名
SELECT   tablename   FROM   pg_tables;

# 创建新表 
CREATE TABLE user_tbl(name VARCHAR(20), signup_date DATE);

# 插入数据 
INSERT INTO user_tbl(name, signup_date) VALUES('张三', '2013-12-22');

# 选择记录 
SELECT * FROM user_tbl;

# 更新数据 
UPDATE user_tbl set name = '李四' WHERE name = '张三';

# 删除记录 
DELETE FROM user_tbl WHERE name = '李四' ;

# 添加栏位 
ALTER TABLE user_tbl ADD email VARCHAR(40);

# 更新结构 
ALTER TABLE user_tbl ALTER COLUMN signup_date SET NOT NULL;

# 更名栏位 
ALTER TABLE user_tbl RENAME COLUMN signup_date TO signup;

# 设置字段默认值（注意字符串使用单引号）
ALTER TABLE user_tbl ALTER COLUMN email SET DEFAULT 'example@example.com';

# 去除字段默认值
ALTER TABLE user_tbl ALTER email DROP DEFAULT;

# 删除栏位 
ALTER TABLE user_tbl DROP COLUMN email;

# 表格更名 
ALTER TABLE user_tbl RENAME TO backup_tbl;

# 删除表格 
DROP TABLE IF EXISTS backup_tbl;

```

# 参考文献

- [原文地址](http://www.ruanyifeng.com/blog/2013/12/getting_started_with_postgresql.html)
