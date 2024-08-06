---
title: Linux安装配置NFS
date: 2023-4-18
tags:
  - Linux
  - CentOS
  - NFS
categories: 
- 运维
- NFS
keywords: 'Linux,CentOS,NFS'
cover: https://qiufuqi.github.io/img/hexo/20230417142330.png
abbrlink: centos_nfs
comments: false
---

两台CentOS7虚拟机，分别记为A（服务端10.128.0.96） B（客户端192.168.0.101）

# 服务端
## 安装nfs
在A服务端安装对应nfs服务，并修改对应端口
``` bash
[root@localhost ~]# yum -y install rpcbind nfs-utils
[root@localhost ~]# vi /etc/sysconfig/nfs
 
# 设置各种*port=...参数
# TCP port rpc.lockd should listen on.
LOCKD_TCPPORT=32803
# UDP port rpc.lockd should listen on.
LOCKD_UDPPORT=32769
 
# Port rpc.statd should listen on.
STATD_PORT=662
# Outgoing port statd should used. The default is port
# is random
STATD_OUTGOING_PORT=2020
```
## 开机自启
设置开机启动并启动对应服务
``` bash
开机自动启动
systemctl enable rpcbind
systemctl enable nfs
systemctl enable nfs-lock
systemctl enable nfs-idmap
 
启动服务
systemctl start rpcbind
systemctl start nfs
systemctl start nfs-lock
systemctl start nfs-idmap
```
## 端口管理
查看端口占用，并通过防火墙 或者暂时关闭防火墙
``` bash
[root@localhost ~]# rpcinfo -p
   program vers proto   port  service
    100000    4   tcp    111  portmapper
    100000    3   tcp    111  portmapper
    100000    2   tcp    111  portmapper
    100000    4   udp    111  portmapper
    100000    3   udp    111  portmapper
    100000    2   udp    111  portmapper
    100005    1   udp  20048  mountd
    100005    1   tcp  20048  mountd
    100005    2   udp  20048  mountd
    100005    2   tcp  20048  mountd
    100005    3   udp  20048  mountd
    100005    3   tcp  20048  mountd
    100003    3   tcp   2049  nfs
    100003    4   tcp   2049  nfs
    100227    3   tcp   2049  nfs_acl
    100003    3   udp   2049  nfs
    100003    4   udp   2049  nfs
    100227    3   udp   2049  nfs_acl
    100021    1   udp  55264  nlockmgr
    100021    3   udp  55264  nlockmgr
    100021    4   udp  55264  nlockmgr
    100021    1   tcp  43310  nlockmgr
    100021    3   tcp  43310  nlockmgr
    100021    4   tcp  43310  nlockmgr
    100024    1   udp  47115  status
    100024    1   tcp  47466  status
 
开启以下端口
[root@localhost ~]# firewall-cmd --zone=public --add-port=111/tcp --permanent
[root@localhost /]# firewall-cmd --zone=public --list-ports
20048/udp 20048/tcp 2049/udp 2049/tcp 111/udp 111/tcp
```
## 设置共享目录
设置需要共享的目录（以/home目录为例），并加载
``` bash

[root@localhost ~]# vi /etc/exports

#填入以下内容 ip为B客户端网段
/home 192.168.0.101/24(rw,root_squash,all_squash,sync,anonuid=1000,anongid=1000)
#或者是网段
/home 192.168.0.0/24(rw,sync)
 
# 每次添加新的IP段，需要执行以下命令重新挂载
[root@localhost ~]# exportfs -r
```

# 客户端
## 挂载硬盘
挂载分享的硬盘，可以看到已经挂载成功
``` bash
[root@localhost /]# mkdir /data
[root@localhost /]# mount -t nfs 10.128.0.96:/home /data
[root@localhost /]# df -h
Filesystem               Size  Used Avail Use% Mounted on
devtmpfs                 1.9G     0  1.9G   0% /dev
tmpfs                    1.9G     0  1.9G   0% /dev/shm
tmpfs                    1.9G  8.5M  1.9G   1% /run
tmpfs                    1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/mapper/centos-root   50G  1.4G   49G   3% /
/dev/vda1               1014M  150M  865M  15% /boot
/dev/mapper/centos-home   46G   33M   46G   1% /home
tmpfs                    379M     0  379M   0% /run/user/0
10.128.0.96:/home         46G   32M   46G   1% /data
```
## 自动挂载
开机自动挂载
``` bash
[root@localhost /]# vi /etc/rc.local
# 添加以下
mount -t nfs 10.128.0.96:/home /data
[root@localhost ~]# chmod +x /etc/rc.local
```

# 问题处理
搭建好nfs服务后，在client端进行挂载时，提示：
mount.nfs: access denied by server while mounting 192.168.0.124:/server/tools/repo
出现此类错误原因大致为：
- 权限问题
- 防火墙机制问题
- 共享配置文件问题

[参考解决](https://www.cnblogs.com/su-root/p/11355486.html)