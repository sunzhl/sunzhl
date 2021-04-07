---
title: Linux 下配置MySQL多实例[mysqld_multi]
date: 2021-03-06 21:14:23
categories: MySQL
tags: MySQL
---

## 一、MySQL多实例介绍

### 1.1.什么是MySQL多实例

MySQL多实例就是在一台机器上开启多个不同的服务端口（如：3306,3307），运行多个MySQL服务进程，通过不同的socket监听不同的服务端口来提供各自的服务；

### 1.2.MySQL多实例的特点有以下几点

1. 有效利用服务器资源，当单个服务器资源有剩余时，可以充分利用剩余的资源提供更多的服务。
2. 节约服务器资源
3. 资源互相抢占问题，当某个服务实例服务并发很高时或者开启慢查询时，会消耗更多的内存、CPU、磁盘IO资源，导致服务器上的其他实例提供服务的质量下降；

<!--more--> 

### 1.3.部署mysql多实例的两种方式

第一种是使用多个配置文件启动不同的进程来实现多实例，这种方式的优势逻辑简单，配置简单，缺点是管理起来不太方便；
第二种是通过官方自带的`mysqld_multi`使用单独的配置文件来实现多实例，这种方式定制每个实例的配置不太方面，优点是管理起来很方便，集中管理；

### 1.4.同一环境下安装两个数据库，必须处理以下问题

- 配置文件安装路径不能相同
- 数据库目录不能相同
- 启动脚本不能同名
- 端口不能相同
- `socket`文件的生成路径不能相同


## 二、多实例部署

### 2.1.安装MySQL


> 说明：此处使用的是面编译的二进制包安装，版本为 `mysql-5.7.27-linux-glibc2.12-x86_64.tar.gz`

#### 2.1.1 创建用户

```bash
shell> groupadd mysql  ## 创建`mysql`用户组 
shell> useradd -r -g mysql -s /bin/false mysql ## 创建 `mysql` 用户
```
#### 2.1.2 解压文件

```bash
shell> cd /usr/local ## 
shell> tar zxvf mysql-5.7.27-linux-glibc2.12-x86_64.tar.gz ## 解压文件
shell> ln -s mysql-5.7.27-linux-glibc2.12-x86_64.tar.gz mysql ## 创建软连接
```
#### 2.1.3 创建相关目录并分配权限

```bash
shell> cd mysql
shell> mkdir data/mysql_3306 #创建3306实例目录
shell> mkdir data/mysql_3307 #创建3307实例目录

shell> chown mysql:mysql data/mysql_3306 -R #更改目录权限
shell> chown mysql:mysql data/mysql_3307 -R #更改目录权限

```
> 说明：此处分别为实例 `3306`和`3307`创建了两个目录，用了存放数据库表文件，同时把目前权限更为给了 `mysql`用户。

#### 2.1.4 配置环境变量

```bash
echo 'export PATH=$PATH:/usr/local/mysql/bin' >>  /etc/profile 
source /etc/profile 
```
>注意：
>切记需要把`mysql` 加入到环境变量中，否则 在启动时出现如下错误：
>WARNING: my_print_defaults command not found.
>Please make sure you have this command available and
>in your path. The command is available from the latest
>MySQL distribution.
>`ABORT: Can't find command 'my_print_defaults'.`
>This command is available from the latest MySQL
>distribution. Please make sure you have the command
>in your PATH.

#### 2.1.5 更改配置文件

创建文件 ` /etc/my.cnf` 内容如下所示：
```bash
[mysqld_multi]
mysqld     = /usr/local/mysql/bin/mysqld_safe
mysqladmin = /usr/local/mysql/bin/mysqladmin
user       = root
pass       = root@123
log        = /usr/local/mysql/log/mysql_multi.log

[mysqld3306]
basedir    = /usr/local/mysql  
datadir    = /usr/local/mysql/data/mysql_3306
socket     = /tmp/mysql_3306.sock
port       = 3306
pid-file   = /usr/local/mysql/data/mysql_3306/mysql.pid
user       = mysql
log-output          = file
slow_query_log      = 1
long_query_time     = 1
slow_query_log_file = /usr/local/mysql/log/3306/slow.log
log-error           = /usr/local/mysql/log/3306/error.log
binlog_format       = mixed
server_id           = 1
log_bin             = mysql-bin

[mysqld3307]
basedir    = /usr/local/mysql
datadir    = /usr/local/mysql/data/mysql_3307
socket     = /tmp/mysql_3307.sock
port       = 3307
pid-file   = /usr/local/mysql/data/mysql_3307/mysql.pid
user       = mysql
server_id  = 2
slow_query_log      = 1
long_query_time     = 1
slow_query_log_file = /usr/local/mysql/log/3307/slow.log
log-error           = /usr/local/mysql/log/3307/error.log

```
> 说明：
> 1. 可以通过命令 `mysqld_multi --example` 查看多实例配置文件示例。
> 2. `[mysqld_multi]` 中的`user`和`pass`分别为数据库示例的登录用户名和密码，多实例直接要配置相同，否则 执行 `mysqld_multi stop` 命令是无法停止服务。
> 3. `[mysqldN]` 表示示例组，必须以`msyqld`开通后面的`N`为整数。
> 4. `socket` 指定的最好指定到 `/tmp` 目录下。

#### 2.1.6 初始化数据库

1. 初始化`3306`实例
```bash
 bin/mysqld --defaults-file=/etc/my.cnf --initialize --user=mysql --basedir=/usr/local/mysql/ --datadir=/usr/local/mysql/data/mysql_3306
```
执行上述命令后，如下图所示，红色框中为初始化密码，<span style="color:red;font-weight:bold">要记住</span>
![](2021/03/06/mysqld-multi/20210305171659975.png)

3. 初始化`3307`实例
`3307`实例与`3306`实例类似，只是把 `--datadir` 属性改为 `3307`实例的路径即可
```bash
bin/mysqld --defaults-file=/etc/my.cnf --initialize --user=mysql --basedir=/usr/local/mysql/ --datadir=/usr/local/mysql/data/mysql_3307
```
> 注意：
> 在初始化是需要把参数 `--defaults-file=/etc/my.cnf` 放到所有参数的前面，否则会报如下错误初始化失败。`unknown variable 'defaults-file=/etc/my.cnf'`

#### 2.1.7 启动实例

1. 查看实例状态
```bash
mysqld_multi report
```
输出结果：
```bash
Reporting MySQL servers
MySQL server from group: mysqld3306 is not running
MySQL server from group: mysqld3307 is not running
```
> 配置了两个组，都处于 `not running` 状态
2. 启动全部实例
```bash
mysqld_multi start
```
出现如下，说明启动成功：
```bash
Reporting MySQL servers
MySQL server from group: mysqld3306 is running
MySQL server from group: mysqld3307 is running
```
> 说明：
>      如果此处还为 `not running` 说明启动失败，可以查看`[mysqld_multi]`中`log`属性指定的日志文件。
3. 停止所有服务
```bash
mysqld_multi stop
```
4. 其他命令
```bash
 mysqld_multi start 3306   # 启动单个服务
 mysqld_multi report 3306  # 查看单个服务状态
 mysqld_multi stop  3306   # 停止单个服务
```

#### 2.1.8 登录服务

1. 方式一 通过指定 `ip` 和`port`登录
```bash
   mysql -uroot -p'XXXXX' -h127.0.0.1 -P3306
```
2. 方式二通过指定`sock` 文件登录
```bash
mysql -uroot -p'XXXXX' -S /tmp/mysql_3306.sock
```
> 说明： 其中`-p` 中替换为 `2.1.6 初始化数据库`中生成的随机密码。

#### 2.1.9 更改初始密码

登录服务后，执行如下命令更改 `root`用户密码

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'new_password';
```
或者
```sql
 set password for root@'localhost'=password('new_password');
 flush privileges;
```

### 2.2 常见问题

<span style="color:red;font-weight:bold">2.2.1 ABORT: Can't find command 'my_print_defaults'.</span>

   此问题是没有把`mysql`加入到环境变量中。参考  `2.1.4 配置环境变量`

<span style="color:red;font-weight:bold">2.2.2 unknown variable 'defaults-file=/etc/my.cnf'</span>

   此问题是在初始化时，需要把 ` --defaults-file` 放到所有参数的最前面。

<span style="color:red;font-weight:bold">2.2.3 `mysqld_multi stop` 无效问题</span>

1. 把配置文件`/etc/my.cnf`中的`[mysqld_multi]` 模块中`password`改为`pass`
2. 查看配置文件`/etc/my.cnf`中的`[mysqld_multi]`模块中配置的 `user` 和`pass`属性是否跟数据库实例`用户名`和`密码`一致。

<span style="color:red;font-weight:bold">2.2.4 No groups to be reported (check your GNRs)</span>

1. 查看配置文件`my.cnf`是否在`/etc`目录下。如果不在目录下可以指定文件位置。如下：

```bash
mysqld_multi --defaults-file=/usr/local/mysql/my.cnf [start|report]
```

2. 检查配置文件`my.cnf`配置是否存在错误

<span style="color:red;font-weight:bold">2.2.4 ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock'</span>

此问题是由于连接`mysql`服务的用户，没有访问`mysqld.sock`文件的权限，给用户分配权限即可。