---
title: Docker安装Nginx
date: 2022-09-26
tags:
  - Linux
  - Docker
  - Nginx
categories: 
- 运维
- Docker
- Nginx
keywords: 'Linux,Docker,Nginx'
cover:  https://qiufuqi.github.io/img/hexo/20221026164307.png
abbrlink: docker_nginx
comments: false
---

Docker安装Nginx
前提：部署好[docker环境](/docker_install)
关闭selinux以及[防火墙（或放行端口）](centos_firewalld)

## 查找Nginx镜像
[dockerHub官方地址](https://registry.hub.docker.com/)
在上方搜索栏里输入nginx
![](https://qiufuqi.github.io/img/hexo/20221026170934.png)
找到要拉取的镜像版本，在tag下找到版本
![](https://qiufuqi.github.io/img/hexo/20221026171052.png)

或者使用命令行查询
``` bash
docker search nginx
```
## 拉取nginx镜像
不指定版本：
``` bash
[root@localhost ~]# docker pull nginx
[root@localhost ~]# docker pull nginx:latest
```
指定版本号：
``` bash
[root@localhost ~]# docker pull nginx:1.20.2
```
![](https://qiufuqi.github.io/img/hexo/20221026171306.png)


``` bash
[root@localhost ~]# docker images
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
nginx        1.20.2    0584b370e957   5 months ago   141MB
```
## 创建nginx实例
-d nginx： 设置容器在在后台一直运行
-v 主机目录：容器目录
--privileged=true 是通过root权限操作
``` bash
# 自动重启可加入：--restart=always
docker run -p 80:80 --name nginx -d nginx:1.20.2

```

## 查询容器状态
``` bash
[root@localhost ~]# docker ps -a
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                               NAMES
448acc833582   nginx:1.20.2   "/docker-entrypoint.…"   45 seconds ago   Up 45 seconds   0.0.0.0:80->80/tcp, :::80->80/tcp   nginx

# 无法启动，查看日志
[root@localhost /]# docker logs 2438d6d7a495

# 查看版本
[root@localhost ~]# docker exec -it 448acc833582 bash
root@448acc833582:/# nginx -v
nginx version: nginx/1.20.2
```




