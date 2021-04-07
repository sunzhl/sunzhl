---
title: CentOS 安装 Nginx
date: 2021-03-10 18:05:46
categories: Nginx
tags: 
  - Nginx
  - CentOS
comments: true
---

### 一、安装依赖包

#### 1.1、 安装gcc-c++

由于`nginx`是采用`c`语言开发的，所以在采用源码安装时需要先进行编译。在`Linux`环境上编译时要用到 `gcc-c++`，所以首先需要先安装或者更新` gcc-c++`;

- 执行命令 

 ```sheel
yum -y install gcc-c++
 ```

#### 1.2、安装pcre  pcre-devel 

由于`nginx` 中 http模块是用 `pcre` 解析正则表达式。所以需要安装`pcre` 库。

- 执行命令

```sheel
yum -y install pcre  pcre-devel
```

#### 1.3、安装 zlib zlib-devel

在`nginx`中需要对响应数据进行压缩，需要用到`zlib`包。

- 执行命令  

```shell
yum -y install zlib zlib-devel**
```
#### 1.4、安装 openssl  openssl-devel

在`nginx`中支持 `https` 协用需要用到 `openssl` 

- 执行命令   
 ```sheel
yum -y install openssl openssl-devel
 ```


### 二、安装 nginx

#### 2.1、官方下载nginx源码包

通过`wget` 下载`nginx`源码包

```shell
wget http://nginx.org/download/nginx-1.15.3.tar.gz
```

#### 2.2、解压
```shell
tar  -zxvf  nginx-1.15.3.tar.gz** 
```
#### 2.3、配置

```shell
cd nginx-1.15.3   #进入解压后的目录，
```
![img](2021/03/10/contos-install-nginx/clipboard-1615372092081.png)

- 执行命令   
```shell
./configure  --prefix=/usr/local/nginx \  #配置目录

​	 --with-debug \                #配置nginx debug方式，在测试环境上可以配置上此项，方便测试调试
	
​	 --with-http_stub_status_module \ #客户端状态
	
​	 --with-http_gzip_static_module \    #gzip模块
	
​	 --with-http_ssl_module  #SSL模块
```

![img](2021/03/10/contos-install-nginx/clipboard-1615372114687.png)

> 说明：
>
> - --with-http_gzip_static_module  # 安装gzip模块，支持 gzip 压缩功能
> - --with-http_ssl_module          # 安装SSL 模块，支持 https 请求
> - --with-http_stub_status_module
>

#### 2.4、编译

- 执行命令进行编译
```shell
   shell> make
```

### 2.4、安装

- 执行命令安装    
```shell
 shell> make install
```

![img](2021/03/10/contos-install-nginx/clipboard-1615372127183.png)

### 2.6、配置环境变量

为了执行方便把 `nginx` 加入到环境变量中。root 用户下打开  **/etc/profile** 修改,在文件末尾添加下配置

```bash
  export NGINX_HOME=/usr/local/nginx
  export PATH=$PATH:$NGINX_HOME/sbin
```
保存后执行 **source /etc/profile**  是配置起作用

### 3、启动、停止 nginx

- nginx            启动nginx
- nginx -s stop    平滑停止 nginx  **此方式停止步骤是待nginx进程处理任务完毕进行停止**
- nginx -s quit    强制停止nginx    **此方式相当于先查出nginx进程id再使用kill命令强制杀掉进程**
- nginx -s reload   相对于重启，把配置文件重新加载
- nginx  -t         检查配置文件是否正确。