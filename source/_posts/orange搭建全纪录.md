---
title: orange搭建全纪录
date: 2017-05-02 22:19:42
comments: true
categories:
- nginx
tags:
- nginx
- openresty
- Lua
---



最近开始在研究openresty和一个叫orange的项目，想要实现对服务器请求相应情况的监控和统计。废话不多说，先把环境搭建起来，再慢慢研究。这时搭建orange项目的整个过程和过程中遇到的问题，记录下来，方便以后查阅和参考。

## 安装前准备

cent os 安装nginx依赖库

```sh
yum -y install readline-devel pcre-devel openssl-devel gcc perl
```

## 安装openresty

1. 下载安装
  我们可以到[官网](https://openresty.org/cn/download.html). 找到需要下载的包，右击复制下载链接。
```sh
# 下载
cd /usr/local
wget https://openresty.org/download/openresty-1.11.2.4.tar.gz --no-check-certificate
# 解压
tar xzvf openresty-1.11.2.4.tar.gz
# 下载负载均衡插件
git clone https://github.com/gnosek/nginx-upstream-fair.git
```
2. 编译安装
```sh
cd openresty-1.11.2.4
./configure --prefix=/usr/local/openresty_for_orange --with-luajit --without-http_redis2_module  --with-http_iconv_module --with-http_stub_status_module --with-http_ssl_module --add-module=/usr/local/nginx-upstream-fair --add-module=/usr/local/ngx_http_dyups_module
# 如果出现
Type the following commands to build and install:
make 或者gmake
make install 或者 gmake install
# 表示配置成功
# 继续执行
make & make install
或者
gmake & gmake install
```
3. 设置环境变量

  我们需要将nginx和resty命令配置到环境变量中，在orange中需要使用到。
```sh
# 配置环境变量
vim /etc/profile
# 添加
export PATH=$PATH:/usr/local/openresty/nginx/sbin
export PATH=$PATH:/usr/local/openresty/bin
 # 保存退出
source /etc/profile
# 测试
nginx -v
resty -v 
# 是否正常输出版本信息
```
在设置环境变量时，可能会遇到一些问题。如果遇到环境变量设置不起作用时，可能是因为系统原生装有nginx，可以使用命令进行卸载。
```
sudo apt-get --purge remove nginx
```

## 安装lor框架
lor有三种安装方式。
**1）使用脚本安装**
使用Makefile安装lor框架:
```sh
git clone https://github.com/sumory/lor
cd lor
make install
```
默认lor的运行时lua文件会被安装到/usr/local/lor下， 命令行工具lord被安装在/usr/local/bin下。
如果希望自定义安装目录， 可参考如下命令自定义路径：
```sh
make install LOR_HOME=/path/to/lor LORD_BIN=/path/to/lord
```
执行**默认安装**后, lor的命令行工具lord就被安装在了/usr/local/bin下, 通过which lord查看:
```
$ which lord
/usr/local/bin/lord
```
lor的运行时包安装在了指定目录下, 可通过lord path命令查看。
**2）使用opm安装(推荐)**
opm是OpenResty即将推出的官方包管理器，从v0.2.2开始lor支持通过opm安装：
```sh
cd /usr/local/openresty/bin
./opm install sumory/lor
```
注意： 目前opm不支持安装命令行工具，所以此种方式安装后不能使用lord
命令。
**3）使用homebrew安装**
除使用以上方式安装外, Mac用户还可使用homebrew来安装lor, 该方式由[@syhily](https://github.com/syhily)提供， 更详尽的使用方法请参见[这里](https://github.com/syhily/homebrew-lor)。
```
$ brew tap syhily/lor$ brew install lor
```
至此， lor框架已经安装完毕，如果是使用第一种方法安装可以使用如下命令检查是否安装成功。
```sh
lord -h
lor ${version}, a Lua web framework based on OpenResty.
```
## Orange安装
首先拷贝orange项目到服务器本地
```sh
git clone https://github.com/sumory/orange.git
```
**配置mysql**
- 在MySQL中创建数据库，名为orange
- 将与当前代码版本配套的SQL脚本(如install/orange-v0.6.3.sql)导入到orange库中

**修改配置文件**
[参考github readme说明](https://github.com/sumory/lor/blob/master/README_zh.md)

**启动**
```sh
cd /usr/local/orange
sh start.sh
```



## 其他

在使用ngx_http_dyups_module模块之前，需要在 nginx.conf中添加配置：

```jso
server {
        listen 8081;
        location / {
            dyups_interface;
        }
    }
```

否则会出错，无法启动nginx。