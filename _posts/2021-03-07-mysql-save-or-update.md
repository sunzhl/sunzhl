---
title: MySQL 没有新增，存在更新
date: 2020-05-09 11:20:50
categories: MySQL
tags: MySQL
---

### 一、前言

 在项目开发中会经常遇到，在插入数据库时，当不存在时就新增此条数据，当存在则更新本条数据。最近项目中也遇到了此类问题，在`MySQL`主要提供了两种方式，现将使用方式，及却别记录一下，加强记忆。

### 二、示例表结构

```sql
-- 创建表结构
DROP TABLE IF EXISTS `test`;
CREATE TABLE `test` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT, -- 自增主键
  `number` varchar(3) NOT NULL,                  -- 唯一索引
  `data` varchar(64) DEFAULT NULL,
  `ts` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `number` (`number`)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8;
 
-- 插入一条数据
INSERT INTO `test` ( `number`, `data`, `ts`) VALUES ('123', 'Old', '2020-05-06 11:09:11');
```

<!--more-->

### 三、INSERT ... ON DUPLICATE KEY UPDATE

1. 存在数据更新

   执行语句

   ```sql
   INSERT INTO `test` ( `number`, `data`, `ts`)  VALUES  ('123', 'New', '2020-05-09 09:09:11')
    ON DUPLICATE KEY UPDATE  `data` = values(`data`), `ts` = values(`ts`);
   ```

   返回结果

   ```shell
   Query OK, 2 rows affected (0.09 sec)  --- 执行结果 注意返回结果为 **2** 。这是由于在存在更新是
   ```

   > 说明：
   >
   > 由于` number=123`已经存在，所以在执行`INSERT`操作后，更新数据。

   

2. 不存在数据新增

   执行语句

   ```sql
   INSERT INTO `test` ( `number`, `data`, `ts`)  VALUES  ('abc', '张三', '2020-05-09 09:09:11')
    ON DUPLICATE KEY UPDATE  `data` = values(`data`), `ts` = values(`ts`);
   ```

   查询结果

   ![img](2020/05/09/mysql-save-or-update/20200509095400477.png)

> 说明：执行 INSERT ... ON DUPLICATE KEY UPDATE 语句返回值情况
>
> 1、如果插入一新行只执行后返回值为 1；
>
> 2、如果执行了UPDATE语句返回值为 2；
>
> 3、如果插入值存在，但是数据和表中对应行的值一样，没有变化，则返回值为 0；

### 四、REPLACE

1. 存在数据更新

   执行语句

   ```sql
   REPLACE INTO test(`number`, `data`, `ts`) VALUES ('456', 'replace', CURRENT_TIMESTAMP);
   ```

   

2. 不存在新增

   执行语句

   ```sql
   REPLACE INTO test(`number`, `data`, `ts`) VALUES ('789', '插入操作', CURRENT_TIMESTAMP);
   ```

>  说明：
>
>  1、如果数据表中不存在主键或者唯一索引相同的数据则增加一条返回条数为 1；
>
>  2、如果数据表中存在主键或唯一索引相同的数据无论数据是否完全相同都会更新，且返回 条数 为 2；
>
>  3、由于replace执行的是先 delete操作 在执行 insert 操作，索引如果存在自增主键，主键会有变化。而 第一种情况自增主键不会变化。



### 五、要求

这两种方式都需要有`唯一索引`或者`主键`。如果表中不存在 主键或者唯一索引，则其功能相当于 insert语句。