---
title: Keepalived安装部署
date: 2022-10-22
tags:
  - Linux
  - keepalive
categories: 
- 运维
- Nginx
- keepalived
keywords: 'Linux,keepalived'
cover: https://qiufuqi.github.io/img/hexo/20221108155048.png
abbrlink: keepalive_install
comments: false
---

**Keepalived安装部署**
Keepalived 2.2.7 安装
安装方式如下：
- YUM安装
- 编译安装
- docker安装

# YUM安装
yum安装keepalived版本不好控制
## yum安装
``` bash
[root@master-node ~]# yum -y install keepalived
```
## 查看版本
``` bash
[root@master-node ~]# rpm -qa|grep keepalived
keepalived-1.3.5-19.el7.x86_64
```
查看安装后目录
``` bash
[root@master-node ~]# rpm -qc keepalived
/etc/keepalived/keepalived.conf
/etc/sysconfig/keepalived
```

# 编译安装
## 获取安装包
下载[安装包](https://www.keepalived.org/download.html)并解压
``` bash
[root@master-node opt]# cd /opt
[root@master-node opt]# wget https://www.keepalived.org/software/keepalived-2.2.7.tar.gz --no-check-certificate
[root@master-node opt]# tar -zxvf keepalived-2.2.7.tar.gz
```
## 编译安装
安装在/etc/keepalived目录下
``` bash
[root@master-node ~]# yum install -y make gcc gcc-c++ openssl openssl-devel
[root@master-node ~]# cd /opt/keepalived-2.2.7
[root@master-node keepalived-2.2.7]# ./configure --prefix=/etc/keepalived --sysconf=/etc
[root@master-node keepalived-2.2.7]# make && make install

#让系统识别nginx的操作命令
[root@slave-node nginx-1.20.2]# ls -s /etc/keepalived/sbin/keepalived /usr/local/sbin/
```
## 添加系统服务
如果已存在则不需要更改
``` bash
[root@master-node keepalived]# vi /lib/systemd/system/keepalived.service
[Unit]
Description=LVS and VRRP High Availability Monitor
After=network-online.target syslog.target
Wants=network-online.target
Documentation=man:keepalived(8)
Documentation=man:keepalived.conf(5)
Documentation=man:genhash(1)
Documentation=https://keepalived.org

[Service]
Type=forking
PIDFile=/run/keepalived.pid
KillMode=process
EnvironmentFile=-/etc/sysconfig/keepalived
ExecStart=/etc/keepalived/sbin/keepalived  $KEEPALIVED_OPTIONS
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
```

## 启动keepalived
``` bash
[root@master-node ~]# systemctl start keepalived
[root@master-node ~]# systemctl enable keepalived
[root@master-node sbin]# ps -ef|grep keepalived
root     15388  9182  0 16:02 pts/0    00:00:00 grep --color=auto keepalived
```
## 常见错误
- systemctl start keepalived启动异常
Failed to start LVS and VRRP High Availability Monitor.
缺失配置文件,创建该文件
``` bash
[root@master-node ~]# cd /etc/keepalived/
[root@master-node keepalived]# > keepalived.conf

# 重新启动
[root@master-node keepalived]# systemctl start keepalived
[root@master-node sbin]# cd /etc/keepalived/sbin/ && ./keepalived
```