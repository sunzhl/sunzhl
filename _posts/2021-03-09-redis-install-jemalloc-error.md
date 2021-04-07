---
title: redis 编译报致命错误：jemalloc/jemalloc.h：没有那个文件或目录
date: 2021-03-09 09:18:41
categories: Redis
tags: 
 - Redis
 - jemalloc
---


## 一、问题

在 `centOS7`环境下安装`redis-5.0.4`时在编译阶段遇到了<span style="color: #FF0000; font-weight: bold">致命错误：jemalloc/jemalloc.h：没有那个文件或目录</span>。

## 二、原因分析

在`Redis`的`README.md`有如下一段话：

> Allocator
>---------
>
>Selecting a non-default memory allocator when building Redis is done by setting
>the `MALLOC` environment variable. Redis is compiled and linked against libc
>malloc by default, with the exception of jemalloc being the default on Linux
>systems. This default was picked because jemalloc has proven to have fewer
>fragmentation problems than libc malloc.
>
>To force compiling against libc malloc, use:
>
 >   % make MALLOC=libc
>
>To compile against jemalloc on Mac OS X systems, use:
>
>    % make MALLOC=jemalloc


说关于分配器 `allocator`， 如果有`MALLOC`  这个环境变量,会有用这个环境变量的 去建立Redis。而且`libc` 并不是默认 的分配器，默认的是 `jemalloc`, 因为`jemalloc`被证明比`libc`有更少的` fragmentation problems `。但是如果你又没有`jemalloc` 而只有` libc` 当然 `make` 出错。 所以加这么一个参数。

<!--more-->

## 三、解决方法

### 1、`make` 时指定分配器为`libc`

```shell
make MALLOC=libc
```

### 2、安装`jemalloc`分配器

**1. 安装`jemalloc`**

```shell
wget https://github.com/jemalloc/jemalloc/releases/download/5.0.1/jemalloc-5.0.1.tar.bz2
tar -jxvf jemalloc-5.0.1.tar.bz2
cd jemalloc-5.0.1
yum install autogen autoconf
 
./autogen.sh
make -j2
make install
ldconfig
cd ../
rm -rf jemalloc-5.0.1 jemalloc-5.0.1.tar.bz2
```

**2. 重新编译**

> 首先删除之前已经解压的 `redis` 包，重新解压。然后在执行 `make` 和 `make install` 即可。
    