---
title: 配置MySQL开机自启动
date: 2021-03-08 15:13:32
categories: MySQL
tags: 
 - MySQL
 - 自启动
 - Linux
---



### 一、概述

在`Linux`环境下配置完多实例`MySQL`后，每次开机都要手动启动，现配置成开机自启动模式。在多实例情况下命令`chkconfig `不再起作用，所有就需要我们手动配置了。

### 二、Linux启动小知识

在配置`MySQL` 多实例启动时，首先让我们了解一下，`Linux`启动的知识点。正常情况下`Linux`的启动顺序如下：

1. 加载内核
2. 执行`init`程序
3. `/etc/rc.d/rc.sysinit`  有`init` 执行的第一个脚本
4. `/etc/rc.d/rc $RUNLEVEL`   $RUNLEVEL为缺省的运行模式，`linux` 共有7种运行模式。
5.  `/etc/rc.d/rc.local`  相应级别服务启动之后、在执行该文件(其实也可以把需要执行的命令写到该文件中)
6. `/sbin/mingetty`   等待用户登录

这里有好几种方法，我们选择第五条，即将执行命令加到`/etc/rc.d/rc.local` 中。

<!--more-->

### 三、Ubuntu18.04  环境

在`ubuntu18.04 `下不在使用`initd`管理系统，改为使用`systemd`。然而 ，`systemd`改变比较大，比较难用。使用`systemd`设置开机启动，为了像以前一样，在`/etc/rc.local`中设置开机启动程序。

#### 1、实现原理

`systemd`默认会读取`/etc/systemd/system`目录下的配置文件，该目录下的文件实质是链接的`/lib/systemed/system` 下的文件。一般系统安装完`/lib/systemed/system`下会有`rc-local.service`文件，即我们需要配置的文件。`/lib/system/system`目录下的文件如下所示：

```bash
shell > cd /lib/systemd/system
shell > ll
```

输出结果如下所示，找到红框中标出的`rc-local.service`

![image-20210308155726732](2021/03/08/mysql-auto-start/image-20210308155726732.png)

接下来在查看`/etc/systemd/system`目录下的文件。

```bash
shell > cd /etc/systemd/system
shell > ll
```

输出结果如下所示：

![image-20210308160046546](2021/03/08/mysql-auto-start/image-20210308160046546.png)

我们看到此时`/etc/systemd/system`目录下是不存在`rc-local.service`文件的。那么接下来创建链接及配置文件。

#### 2、创建链接

- 执行如下命令，创建链接

  ```shell
  shell > ln -fs /lib/systemd/system/rc-local.service /etc/systemd/system/rc-local.service
  ```

- 查看文件内容

  ```shell
  shell > cd /etc/systemd/system/ 
  shell > cat rc-local.service
  ```

  输出结果如下：

  ```bash
  #  This file is part of systemd.
  #
  #  systemd is free software; you can redistribute it and/or modify it
  #  under the terms of the GNU Lesser General Public License as published by
  #  the Free Software Foundation; either version 2.1 of the License, or
  #  (at your option) any later version.
  
  # This unit gets pulled automatically into multi-user.target by
  # systemd-rc-local-generator if /etc/rc.local is executable.
  [Unit] #启动顺序与依赖关系。
  Description=/etc/rc.local Compatibility
  ConditionFileIsExecutable=/etc/rc.local #指定了执行的文件，
  After=network.target # `network.target` 这个target后面进行执行。也就是网络启动完成之后，执行` /etc/rc.local` 文件。
  
  [Service] # 启动行为,如何启动，启动类型。
  Type=forking
  ExecStart=/etc/rc.local start
  TimeoutSec=0
  RemainAfterExit=yes
  ```

- 添加 [Install]块

  在上述文件的末尾加上如下内容：

  ```bash
  [Install] #定义如何安装这个配置文件，即怎样做到开机启动。
  WantedBy=multi-user.target
  Alias=rc-local.service
  ```

#### 3、修改`/etc/rc.local`文件

在`/etc/rc.local`文件的`exit 0` 之前加入如下内容：

```bash
# 指定 MySQL 的环境变量
MYSQL_HOME=/usr/local/mysql
export MYSQL_HOME
PATH=$PATH:$MYSQL_HOME/bin
export PATH

# 配置启动项
/usr/local/mysql/bin/mysqld_multi --defaults-file=/usr/local/mysql/etc/my.cnf start &

```

> 说明：
>
> 1. 如果 `/etc/` 目录下不存在`rc.local`文件可以自己创建。并赋予可执行权限
> 2. 由于 `/etc/rc.local`文件早于`/etc/profile`文件执行，在`/etc/profile`文件中配置的环境变量还未起作用，所有再次需要配置上 `MySQL` 的环境变量。

#### 4、重启系统

执行`reboot`命令重启系统，查看服务是否启动成功。

执行如下命令，查看是否成功启动：

```shell
shell >  netstat -ln | grep 330
```

输出如下结果表示启动成功：

![image-20210308164210305](2021/03/08/mysql-auto-start/image-20210308164210305.png)



> 说明：
>
> ​     如 `MySQL` 没有正常启动，可以查看日志文件`/var/log/syslog`。从中找到错误日志，进行相应的修改即可。

### 四、CentOS7环境下

在 `CentOS7`环境下配置相对比较简单。只需要在 `/etc/rc.d/rc.local`文件中添加上需要启动的命令，已经给`/etc/rc.d/rc.local`文件赋予可执行权限即可，具体操作如下所示：

#### 1、查看文件

查看`/etc/rc.d/rc.local`文件的初始内容如下：

```shell
shell > cat /etc/rc.d/rc.local
```

 输出内容如下：

```bash
#!/bin/bash
# THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
#
# It is highly advisable to create own systemd services or udev rules
# to run scripts during boot instead of using this file.
#
# In contrast to previous versions due to parallel execution during boot
# this script will NOT be run after all other services.
#
# Please note that you must run 'chmod +x /etc/rc.d/rc.local' to ensure
# that this script will be executed during boot.

touch /var/lock/subsys/local

```

#### 2、添加内容

在`/etc/rc.d/rc.local` 文件的末尾加入要执行的命令，如下所示：

```bash
# 指定 MySQL 的环境变量
MYSQL_HOME=/usr/local/mysql
export MYSQL_HOME
PATH=$PATH:$MYSQL_HOME/bin
export PATH

/usr/local/mysql/bin/mysqld_multi --defaults-file=/etc/my.cnf start &
```

#### 3、赋予可执行权限

```shell
shell > chmod +x /etc/rc.d/rc.local
```

#### 4、重启系统

执行`reboot`命令重启系统，查看服务是否启动成功。

执行如下命令，查看是否成功启动：

```shell
shell >  netstat -ln | grep 330
```

输出如下结果表示启动成功：

![image-20210308164210305](2021/03/08/mysql-auto-start/image-20210308164210305.png)

> 说明：
>
> ​     如 `MySQL` 没有正常启动，可以通过 `systemctl status rc-local.service` 查看启动日志，从中找到错误日志，进行相应的修改即可。



### 五、总结

本篇内容主要介绍了 `Linux` 系统启动流程， `Ubuntu`  和 `centOS` 系统下配置自启动的方式。 以上方式可以配置任何应用为开机自启动，比如可以配置 `redis` 、`tomcat`、`nginx` 等。