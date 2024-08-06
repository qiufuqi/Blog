---
title: Docker安装
date: 2022-09-13
tags:
  - Linux
  - Docker
categories: 
- 运维
- Docker
keywords: 'Linux,Docker'
cover: https://qiufuqi.github.io/img/hexo/20220913163312.png
abbrlink: docker_install
comments: false
---

Docker环境安装（CentOS 7系统）

DockerCE 社区免费版，可永久免费使用；
DockerEE 企业版，功能更全，更强调安全，但需要付费使用；

## 查看本机是否安装
``` bash
yum list installed|grep docker
```

## 移除旧版本
``` bash
yum remove docker  docker-client  docker-client-latest  docker-common  docker-latest docker-latest-logrotate  docker-logrotate  docker-engine
```

## 配置Docker Repository
``` bash
yum install -y yum-utils

yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# 国内ali镜像
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
默认安装stable版本，也可以安装其他版本
- 稳定版本：stable 一般使用此版本
- 预发布版本：test
- 待发布版本：nightly
``` bash
# 使用 版本
yum-config-manager --enable docker-ce-nightly
yum-config-manager --enable docker-ce-test
# 不使用 版本
yum-config-manager --disable docker-ce-nightly
```

## 安装Docker Engine
- 安装最新版本
``` bash
yum -y install docker-ce docker-ce-cli containerd.io
```
- 选择版本安装
``` bash
yum list docker-ce --showduplicates | sort -r
······
docker-ce.x86_64            3:20.10.9-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.8-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.7-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.6-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.5-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.4-3.el7                     docker-ce-stable

yum -y install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io
```

## 启动和配置
``` bash
systemctl start docker
systemctl enable docker
```