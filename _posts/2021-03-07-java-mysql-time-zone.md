---
title: Java应用和Mysql时间相差8个时区的问题
date: 2019-08-29 17:21:27
categories: MySQL
tags: 
  - MySQL
  - JAVA
comments: true
---



### 一、概述

最近在做项目时发现，在无论在`java`应用中使用` java.util.Date`还是使用` java.time.LocalDateTime`类，获取的当前时间保存到数据库中后，数据库中的时间跟应用中获取的时间相差` 8`个小时。

### 二、排查过程

 首先查看数据库和`java`应用的时区设置，发现时区都没为题，都为`东八区`。困惑了好久 再一次配置 数据库连接池时发现，配置的`JDBC`连接路径中多了 `“serverTimezone=UTC”`,参数，去除此参数后重启再次保存时间` java`应用和数据库中的时间一致了。

<!--more-->

### 三、设置数据库的时区

#### 1、首先查看数据库配置的时区

```shell
mysql> show variables like '%time_zone%';
+------------------+--------+
| Variable_name    | Value  |
+------------------+--------+
| system_time_zone | CST    |
| time_zone        | SYSTEM |
+------------------+--------+
```

>   如果` system_time_zone` 为 `CST` 表示此时数据库中设置的时区`非东八区`。

#### 2、修改时区

1. 命令行方式

   ```sql
   mysql> set global time_zone = '+08:00'; -- 设置全局的时区
   Query OK, 0 rows affected (0.00 sec)
   mysql> set time_zone = '+08:00';       -- 设置当前会话
   Query OK, 0 rows affected (0.00 sec)
   ```

   > 说明：
   >
   > 1. 设置完成后需要重启应用才可以
   > 2. 如果通过命令行设置，如果重启`MySQL` 服务后之前设置的就失效了。

2. 修改配置文件

   在配置文件 `my.conf` (Linux环境) 或 `my.ini` (Window环境) 中添加 ``default-time-zone = '+08:00'`。重启数据库即可。

