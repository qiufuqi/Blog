---
title: Docker安装Mysql
date: 2022-09-11
tags:
  - Linux
  - Docker
  - Mysql
categories: 
- 运维
- Docker
- Mysql
keywords: 'Linux,Docker,Mysql'
cover:  https://qiufuqi.github.io/img/hexo/20221011151146.png
abbrlink: docker_mysql
comments: false
---

Docker安装Mysql
前提：部署好[docker环境](/docker_install)
关闭selinux以及[防火墙（或放行端口）](centos_firewalld)

## 查找Mysql镜像
[dockerHub官方地址](https://registry.hub.docker.com/)
在上方搜索栏里输入mysql
![](https://qiufuqi.github.io/img/hexo/20221011114954.png)
找到要拉取的镜像版本，在tag下找到版本
![](https://qiufuqi.github.io/img/hexo/20221011115047.png)

或者使用命令行查询
``` bash
docker search mysql
```
## 拉取Mysql镜像
不指定版本：
``` bash
[root@localhost ~]# docker pull mysql
[root@localhost ~]# docker pull mysql:latest
```
指定版本号：
``` bash
[root@localhost ~]# docker pull mysql:5.7
[root@localhost ~]# docker pull mysql:5.7.38
```
![](https://qiufuqi.github.io/img/hexo/20221011115833.png)

``` bash
[root@localhost ~]# docker images
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
mysql        5.7.38    459651132a11   2 months ago   429MB
```
## 创建Mysql实例
-v 主机目录：容器目录
/var/lib/mysql    (data目录)
/etc/mysql        (配置目录)
/var/log/mysql    (这个是日志目录)
--privileged=true 是通过root权限操作
``` bash
# 自动重启可加入：--restart=always
docker run -p 3306:3306 --name mysql --privileged=true -v /mydata/mysql/log:/var/log/mysql -v /mydata/mysql/data:/var/lib/mysql -v /mydata/mysql/conf:/etc/mysql/conf.d  -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7.38 --restart=always

# 如果创建时未指定--restart=always ,可通过update 命令设置:
docker update --restart=always 容器名称(或者容器ID)
```
**开机自启**
--restart=always
**权限提升**
--privileged=true
容器有root权限
**配置用户**
-e MYSQL_ROOT_PASSWORD=123456
设置初始化root用户的密码
**指定镜像资源**
-d mysql:5.7.38
-d：以后台方式运行实例
mysql:5.7.38：指定用这个镜像来创建运行实例

## 查询容器状态
``` bash
[root@localhost /]# docker ps -a
CONTAINER ID   IMAGE          COMMAND                  CREATED              STATUS              PORTS                                                  NAMES
2438d6d7a495   mysql:5.7.38   "docker-entrypoint.s…"   About a minute ago   Up About a minute   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp   mysql

# 无法启动，查看日志
[root@localhost /]# docker logs 2438d6d7a495
```
使用配置的账号密码成功连接数据库
![](https://qiufuqi.github.io/img/hexo/20221011145259.png)




