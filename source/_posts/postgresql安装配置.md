---
title: postgresql安装配置
comments: true
date: 2017-08-03 22:52:41
categories: 
- DB
tags: 
- DB
---

最近在研究著名的网关系统kong,  kong使用了postgreSql来作为数据库， 以前从来没有接触过postgreSql, 这次终于有机会了解一下，话说postgreSql在最近一年中表现抢眼，市场占有率也越来越高，增长很快。因此把这次安装postgreSql的过程记录下来，为以后继续研究做个参考。本次是在ubuntu系统上安装的。

## 安装

1. 安装postgresql客户端

   ```shell
   sudo apt-get install postgresql-client
   ```

2. 安装postgresql服务端

   ```shell
   sudo apt-get install postgresql
   ```

   安装成功后会默认生成一个名为postgres的数据库和一个名为postgres的数据库用户，密码是随机的。这里需要注意的是，同时还生成了一个名为postgres的Linux系统用户，密码也是随机的。

3. 修改数据库用户postgres密码

   ```shell
   # sudo -u postgres 表示是用postgres用户登录数据库
   sudo -u postgres psql
   ALTER USER postgres WITH PASSWORD '123456';
   #推出客户端
   \q 
   ```

4. 修改linux系统用户postgres用户密码

   ```shell
   #切换到root用户
   su root
   #删除postgres用户密码
   sudo passwd -d postgres
   #设置postgres用户密码
   sudo -u postgres passwd
   #输入两次密码
   psql2017
   ```

5. 修改PostgresSQL数据库配置实现远程访问

   **pg_hba.conf：**配置对数据库的访问权限；

   **postgresql.conf：**配置PostgreSQL数据库服务器的相应的参数。

   vim /etc/postgresql/9.5/main/postgresql.conf

   ​

##  



> http://www.cnblogs.com/zhangpengshou/p/5464610.html