---
title: Redis 安装
date: 2021-03-13 16:01:26
categories: Redis  
tags: 
  - Redis
  - NoSQL
---

### 一、Redis 安装

`Redis` 的安装方式非常简单，步骤如下所示：

#### 1、下载redis安装包

在 `Redis` [官网](https://redis.io/download)上下载最稳定版本的源码，我们这里安装 `5.0.12` 版本

```shell
shell > wget https://download.redis.io/releases/redis-5.0.12.tar.gz
```

#### 2、解压压缩包

```shell
shell > tar -zxvf redis-5.0.12.tar.gz
```

#### 3、建立软连接

```shell
shell > ln -s redis-5.0.12 redis
```

> 说明：
>
>    这里建立软连接的目的是为了不把 `redis` 目录固定在指定版本上，有利于`Redis` 未来版本升级。

#### 4、编译安装

```shell
shell > make 
```

> 如果此步骤出现异常可以查看 [redis 编译报致命错误](http://yby.ink/2021/03/09/redis-install-jemalloc-error/)

#### 5、安装

```shell
shell > make install  
```

> 说明：
>
> ​    此步是把 是把 redis 安装到 /usr/local/bin 这样可以就可以在任何目录下执行 redis 的命令了。

<!--more-->

### 二、启动

`Redis` 有三种启动方式：默认配置、运行配置、配置文件启动。

#### 1、默认配置

在命令行执行以下命令：

```shell
shell > redis-server
```

执行后出现如下日志：

![image-20210313163602612](2021/03/13/install-redis/image-20210313163602612.png)

#### 2、运行配置启动

上面的方式启动，都是运用的默认配置，不能应用于生产环境，那么一些简单的配置可以在启动是指定。命令模式如下所示：

```shell
shell > redis-server --configKey1 configValue1 --configKey2 configValue2
```

例如，我们在启动是指定端口号为 `6380`, 命令如下：

```shell
shell > redis-server --port 6380
```

![image-20210313164124458](2021/03/13/install-redis/image-20210313164124458.png)

这种方式运行配置，虽然可以自定义配置，但是如果需要修改的配置很多每次启动时，都要指定比较麻烦，那么我们可以通过以下方式，把配置写到文件中。

#### 3、配置文件启动

我们可以把文件配置在 `/opt/redis/redis.conf` 中，然后在启动时，指定此文件。如下所示：

```shell
shell > redis-server /opt/redis/redis.conf
```

### 三、停止

`Redis` 提供了 `shutdown` 命令来停止`Redis`服务，如下停止刚才启动的 `6379` 服务 

```shell
shell > redis-cli -p 6379 shutdown
```

> 说明：
>
> 1. Redis 关闭的过程：断开与客户端的连接、持久化文件生成，是一种优雅的关闭方式。
>
> 2. 除了 shutdown 命令关闭 Redis 服务外，还可以使用 kill 进程号的方式关闭掉 Redis。但是不要简单粗暴的使用 kill -9 强制杀死 Redis 服务， 不但不会做持久化操作，还会造成缓冲区等资源不能被优雅关闭。
>
> 3. shutdown 还有一个参数，代表是否在关闭 Redis 前，生成持久化文件：
>
>    redis-cli shutdown save | nosave