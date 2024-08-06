---
title: Docker基本命令
date: 2022-10-11
tags:
  - Linux
  - Docker
  - bash
categories: 
- 运维
- Docker
- Bash
keywords: 'Linux,Docker,Bash'
cover: https://qiufuqi.github.io/img/hexo/20221011151356.png
abbrlink: docker_bash
comments: false
---

Docker基本命令操作

# docker基本命令
## 查看镜像
docker images
``` bash
[root@localhost ~]# docker images
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
mysql        5.7.38    459651132a11   3 months ago   429MB

##字段说明
REPOSITORY：镜像属于的仓库；
TAG：镜像的标签信息，标记同一个仓库中的不同镜像；
IMAGE ID：镜像的唯一ID 号，唯一标识一个镜像，经过md5方式加密过；
CREATED：镜像创建时间；
VIRTUAL SIZE：镜像大小
```
## 查看所有状态容器
docker ps -a
``` bash
[root@localhost ~]# docker ps -a
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                                                  NAMES
118a45fa8a9d   mysql:5.7.38   "docker-entrypoint.s…"   18 minutes ago   Up 13 minutes   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp   mysql

##字段说明
CONTAINER ID：容器的ID号
IMAGE：加载的镜像
COMMAND ：运行的程序
CREATED ：创建时间
STATUS：当前的状态
PORTS：端口映射
NAMES：名称
```
## docker run 指令
docker run hello-world
- 检测本地有没有该镜像（没有的话直接到docker hub上下载)
- create(将镜像创建为容器)+ start 将创建好的容器运行起来
![](https://qiufuqi.github.io/img/hexo/20221011152006.png)

## 查看版本
两种方式可查看
``` bash
root@localhost ~]# docker -v
Docker version 20.10.18, build b40c2f6
[root@localhost ~]# docker version
Client: Docker Engine - Community
 Version:           20.10.18
 API version:       1.41
 Go version:        go1.18.6
 Git commit:        b40c2f6
 Built:             Thu Sep  8 23:14:08 2022
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true
·········
```
## 查看docker信息
docker info
``` bash
[root@localhost ~]# docker info
Client:
 Context:    default
 Debug Mode: false
 Plugins:
  app: Docker App (Docker Inc., v0.9.1-beta3)
  buildx: Docker Buildx (Docker Inc., v0.9.1-docker)
  scan: Docker Scan (Docker Inc., v0.17.0)

Server:
 Containers: 1
  Running: 1
  Paused: 0
·········
```
![](https://qiufuqi.github.io/img/hexo/20221011152223.png)
## 帮助文档
``` bash
[root@localhost ~]# docker --help
```

# docker镜像操作
## 搜索镜像
默认是在公共仓库找，如果有私有仓库，会在私有仓库找
``` bash
#格式：docker search 关键字
#示例： 
docker search nginx 
docker search centos：7
```
## 下载镜像
``` bash
#格式：docker pull 仓库名称[:标签]
#如果下载镜像时不指定标签，则默认会下载仓库中最新版本的镜像，即选择标签为 latest 标签。
docker pull centos:7
docker pull nginx
```
## 查看镜像列表
``` bash
[root@localhost ~]# docker images
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
mysql        5.7.38    459651132a11   3 months ago   429MB
```
## 获取镜像信息
``` bash
#格式：docker inspect  镜像ID
#示例：查看nginx镜像信息
docker inspect 118a45fa8a9d
```
## 添加镜像标签
会多一个镜像，id相同
``` bash
#格式：docker tag 名称:[旧标签] 新名称:[新标签]
#示例：
docker tag nginx：latest nginx:lnmp  #给nginx打上标签lnmp，原来的标签是latest
```
![](https://qiufuqi.github.io/img/hexo/20221011152956.png)
## 删除镜像
``` bash
#格式：
docker rmi 仓库名称:标签	 #当一个镜像有多个标签时，只是删除其中指定的标签
docker rmi 镜像ID号	   #会彻底删除该镜像
```
## 批量删除镜像
``` bash
#docker images -q 可以加载镜像id
 
#批量删除所有镜像
docker rmi `docker images -q`
#批量删除nginx镜像
docker rmi `docker images|grep "nginx"`
```
## 导出\导入镜像
docker save/load
``` bash
#导出镜像
#格式：docker save -o 存储文件名 存储的镜像
docker save -o nginx_v1 nginx:latest			#存出镜像命名为nginx存在当前目录下
docker save -o centos_v1 centos:7 
 
#导入镜像，可以异地导入，但是必须要有docker引擎，并且版本不可以差太多
#格式：docker load < 存出的文件
docker load < nginx_v1 
dokcer load < centos_v1

批量把镜像打包成haha.tar压缩包（但修改名称的镜像包打包不了）
docker save $(docker images | grep -v REPOSITORY | awk 'BEGIN{OFS=":";ORS=" "}{print $1,$2}') -o haha.tar
```
导出镜像
![](https://qiufuqi.github.io/img/hexo/20221011153435.png)
导入镜像
![](https://qiufuqi.github.io/img/hexo/20221011153445.png)

# 容器操作
## 查询所有容器运行状态
``` bash
[root@localhost ~]# docker ps -a
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                                                  NAMES
118a45fa8a9d   mysql:5.7.38   "docker-entrypoint.s…"   37 minutes ago   Up 17 minutes   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp   mysql
```
## 创建容器
docker create
新创建的容器默认处于停止状态，不运行任何程序，需要在其中发起一个进程来启动容器。
``` bash
#格式：docker create [选项] 镜像
#常用选项：
-i：让容器的输入保持打开
-t：让 Docker 分配一个伪终端

#示例：
docker create -it nginx:latest /bin/bash
```
## 启动容器
``` bash
#格式：docker start 容器的ID/名称
docker start 3dcd81095c9f
docker ps -a
```
## 启动容器（一次性执行）
``` bash
#加 -d 选项让 Docker 容器以守护形式在后台运行。并且容器所运行的程序不能结束。
 
#示例1：
docker run --name web1 -itd nginx:latest /bin/bash
 
#示例2：执行后退出
docker run --name test1 centos:7 /usr/local/bash -c ls /   
 
#示例3：执行后不退出，以守护进程方式执行持续性任务
docker run  --name web2 -d centos:7 /bin/bash -c "while true;do echo hello;done" 
```
## 查看容器ip地址
``` bash
#格式：docker inspect 容器id 
docker ps -a   #先查看运行时容器的id
docker inspect 45f2ce7ee445
```
![](https://qiufuqi.github.io/img/hexo/20221011154256.png)
## 进入容器
进入容器的容器状态必须是up状态 和shell 是两种运行模式
- docker run -it会创建前台进程，但是会在输入exit后终止进程。
- docker attach会通过连接stdin，连接到容器内输入输出流，会在输入exit后终止容器进程。
- docker exec -it 会连接到容器，可以像sSH一样进入容器内部，进行操作，可以通过exit退出容器，不影响容器运行。
``` bash
#需要进入容器进行命令操作时，可以使用 docker exec 命令进入运行着的容器。
 
#格式：docker exec -it 容器ID/名称 /bin/bash
-i 选项表示让容器的输入保持打开；
-t 选项表示让 Docker 分配一个伪终端。
-d：守护进程。
 
#示例：进入（三种方式）
docker run -itd centos:7 /bin/bash  #先运行容器
docker ps -a 
①使用run进入，可以使用ctrl+d退出，直接退出终端
docker run -it centos:7 /bin/bash 
 
②想永久性进入，退出后还是运行状态，用docker exec
docker ps -a 
docker exec -it nginx:latest  /bin/bash
 
③docker attach，会通过连接stdin，连接到容器内输入输出流，公在输入exit后终止容器进程（临时性的，不推荐）
```
使用run进入，是一次性进入
![](https://qiufuqi.github.io/img/hexo/20221011154732.png)
永久性进入，用docker exec (退出后，容器仍然会运行)
![](https://qiufuqi.github.io/img/hexo/20221011154750.png)

## 删除容器
不能删除运行状态的容器，只能-f强制删除，或者先停止再删除
已经退出的容器，可以直接删除
``` bash
#格式：docker rm [-f] 容器ID/名称
 
1.#不能删除运行状态的容器，只能-f强制删除，或者先停止再删除
docker rm web1
 
2.#已经退出的容器，可以直接删除
docker rm web2 
 
3.#基于名称匹配的方式删除
docker rm -f web3
 
 4.#删除所有运行状态的容器
docker rm -f `docker ps -q`
 
5.#删除所有容器
docker rm -f `docker ps -aq`
 
6.#有选择性的批量删除 （正则匹配）
docker ps -a | awk ' {print "docker rm "$1}' | bash
 
7.#删除退出状态的容器
for i in `dockef ps -a | grep -i exit | awk '{print $1}' '; do docker rm -f $i;done
```
## 查看docker消耗的资源状态
docker stats
![](https://qiufuqi.github.io/img/hexo/20221011155342.png)

































