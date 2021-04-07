---
title: 解决：timestamp 默认值 ‘0000-00-00 00:00:00’ 报错
date:  2021-01-07 17:15:18
categories: MySQL
tags: MySQL
---



### 一、问题描述

在` mysql5.7` 版本创建如下表结构时出现，`timestamp` 字段默认值错误：如下所示：

```sql
CREATE TABLE `login_log` (
 `id`          bigint(20) UNSIGNED NOT NULL AUTO_INCREMENT ,
 `uuid`        VARCHAR(34) UNIQUE NOT NULL  COMMENT '登录编号',
 `number`      VARCHAR(20) NOT NULL  COMMENT '登录账号',
 `name`        VARCHAR(20) NOT NULL DEFAULT '' COMMENT '登录姓名',
 `login_time`  TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '登录时间',
 `logout_time` TIMESTAMP  COMMENT '退出时间',
 `login_ip`    VARCHAR(15)  COMMENT '登录IP',
 `device`      VARCHAR(15)  COMMENT '设备类型',
 `status`      INT(3) COMMENT '登录状态',
 `message`     VARCHAR(30) COMMENT '登录结果描述',
  PRIMARY KEY (`id`)
)ENGINE=InnoDB DEFAULT CHARACTER SET=utf8 COLLATE=utf8_general_ci COMMENT='登录日志';

[Err] 1067 - Invalid default value for 'logout_time'
```



错误信息为：

> **[Err] 1067 - Invalid default value for 'logout_time'**

<!--more-->

### 二、问题源头 sql_mode

引起这个问题的原因是在没有指定` timestamp` 默认值时，系统是按` ‘0000-00-00 00:00:00’ `处理的，但是在 `sql_mode` 中 模式是不允许设置 此默认值的。我们看下：`sql_mode` 的具体信息，可以使用如下命令查看 当前会话或者全局的 `sql_mode`，如下所示：

```sql
SELECT @@GLOBAL.sql_mode; -- 查看全局模式 

SELECT @@SESSION.sql_mode; -- 查看当前会话的模式
```


查看全局的输出结果如下：

![img](2021/01/07/mysql-timestamp-default/20210107170532886.png)



通过上图中的结果我们可以看到`sql_mode`中有` NO_ZERO_IN_DATE** 和 **NO_ZERO_DATE` 在命令行中输入

### 三、解决方法

1、可以通过命令直接设置，如下所示：

```sql
SET GLOBAL sql_mode = 'modes'; // 设置全局
SET SESSION sql_mode = 'modes'; // 设置当前会话
```

> 说明：
>
> 1. 其中`modes`为: 'STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION'。也就是把 **NO_ZERO_IN_DATE** 和 **NO_ZERO_DATE** 去掉即可。
>
> 2. 如果使用全局的设置完成后需要重新连接数据库。

2、修改配置文件

通过上述命令的方式设置完成后，如果重启数据库，设置的就会失效了。可以通过修改配置文件的方式更改，这样更改完，即使数据库重启也不会失效。设置方式如下：

找到 MySQL 数据库的配置文件，在配置文件中加入如下，然后重启数据库即可。

```
 sql_mode = STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
```



### 四、几种常见 mode 说明

**ONLY_FULL_GROUP_BY**：出现在select语句、HAVING条件和ORDER BY语句中的列，必须是GROUP BY的列或者依赖于GROUP BY列的函数列。

**NO_AUTO_VALUE_ON_ZERO**：该值影响自增长列的插入。默认设置下，插入0或NULL代表生成下一个自增长值。如果用户希望插入的值为0，而该列又是自增长的，那么这个选项就有用了。

**STRICT_TRANS_TABLES**：在该模式下，如果一个值不能插入到一个事务表中，则中断当前的操作，对非事务表不做限制

**NO_ZERO_IN_DATE**：这个模式影响了是否允许日期中的月份和日包含0。如果开启此模式，2016-01-00是不允许的，但是0000-02-01是允许的。它实际的行为受到 strict mode是否开启的影响1。

**NO_ZERO_DATE**：设置该值，mysql数据库不允许插入零日期。它实际的行为受到 strictmode是否开启的影响2。

**ERROR_FOR_DIVISION_BY_ZERO**：在INSERT或UPDATE过程中，如果数据被零除，则产生错误而非警告。如果未给出该模式，那么数据被零除时MySQL返回NULL

**NO_AUTO_CREATE_USER**：禁止GRANT创建密码为空的用户

**NO_ENGINE_SUBSTITUTION**：如果需要的存储引擎被禁用或未编译，那么抛出错误。不设置此值时，用默认的存储引擎替代，并抛出一个异常

**PIPES_AS_CONCAT**：将”||”视为字符串的连接操作符而非或运算符，这和Oracle数据库是一样的，也和字符串的拼接函数Concat相类似

**ANSI_QUOTES**：启用ANSI_QUOTES后，不能用双引号来引用字符串，因为它被解释为识别符。

