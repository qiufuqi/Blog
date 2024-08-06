---
title: Linux虚拟IP配置
date: 2023-4-17
tags:
  - Linux
  - CentOS
  - 虚拟IP
categories: 
- 运维
- 虚拟IP
keywords: 'Linux,CentOS,虚拟IP'
cover: https://qiufuqi.github.io/img/hexo/20230417142330.png
abbrlink: centos_virtual_ip
comments: false
---

# 虚拟IP介绍
虚拟IP地址(VIP) 是一个不与特定计算机或一个计算机中的网络接口卡(NIC)相连的IP地址。数据包被发送到这个VIP地址，但是所有的数据还是经过真实的网络接口。

- 真实IP又被称为管理IP，一般是配置在物理网卡上的实际IP，管理IP是不对外提供用户访问服务的，而作为管理服务器用，如SSH可以通过这个管理IP连接服务器。
- 虚拟IP即VIP，这只是一个概念而已，可能会误导你，实际上就是heartbeat临时绑定在物理网卡上的别名(heartbeat3以上页采用了辅助IP)，你可以在一块网卡上绑定多个别名。这个VIP可以看作是你上网的QQ网名、昵称、外号等。在实际生产环境中，需要在DNS配置中把网站域名地址解析到这个VIP地址，由这个VIP对用户提供服务。如：把www.zhangcong.top解析到VIP 1.1.1.1 上。

# 虚拟IP作用
大部分虚拟ip基本上都用于高可用的架构上边。主机启用虚拟ip，所有访问的请求都会到主机。当主机宕机的时候，高可用软件会将主机的虚拟ip down掉，然后在备机上启用虚拟ip。这样就完成了主备切换。从而保证业务的可用性。

# 创建虚拟IP
在linux中创建虚拟ip有两种方法，分别是：辅助IP和别名IP。 (别名IP将被遗弃，用辅助IP替代)

## 辅助IP
辅助ip是由linux的ip命令去创建和操作的。
### 创建辅助IP
创建命令：ip addr add IP/掩码 dev 网卡名
``` bash
[root@lj-master ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
  ·········
2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
  ·········
[root@lj-master ~]# ip addr add 10.11.7.240/24 dev ens192
```
### 查看辅助ip
使用命令ip a就可以查看，但是不能使用ifconfig –a去查看。
``` bash
[root@lj-master ~]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
·········
2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:8d:75:a2 brd ff:ff:ff:ff:ff:ff
    inet 10.11.7.232/24 brd 10.11.7.255 scope global ens192
       valid_lft forever preferred_lft forever
    inet 10.11.7.240/24 scope global ens192
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:fe8d:75a2/64 scope link 
       valid_lft forever preferred_lft forever
```
### 删除辅助ip
删除命令：ip addr del IP dev 网卡名 (将add改为del即可。)
``` bash
[root@lj-master ~]# ip addr del 10.11.7.240/24 dev ens192
```
### 永久生效
这种方式创建的虚拟ip，可以将生成虚拟ip的命令写到/etc/rc.local中去，开机即可自动加载。

## 别名IP
别名ip是由linux的ifconfig命令去创建和操作的。
### 创建别名IP
创建命令：ifconfig 网关名:1 IP netmask 子网掩码 up
``` bash
[root@lj-master ~]# ifconfig
ens192: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
  ·········
[root@lj-master ~]# ifconfig ens192:1 10.11.7.240 netmask 255.255.255.0 up
```
### 查看别名IP
``` bash
[root@lj-master ~]# ifconfig
ens192: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.11.7.232  netmask 255.255.255.0  broadcast 10.11.7.255
        ·········
ens192:1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.11.7.240  netmask 255.255.255.0  broadcast 10.11.7.255
        ether 00:50:56:8d:75:a2  txqueuelen 1000  (Ethernet)
```
### 删除别名IP
删除命令：ifconfig 网关名:1 IP netmask 子网掩码 down (只要将后边的up改为down就可以了)。
``` bash
[root@lj-master ~]# ifconfig ens192:1 10.11.7.240 netmask 255.255.255.0 down
```
### 永久有效
开机就有虚拟ip，可以在网卡的配置目录中去建立一个新的网卡的配置文件
centos和红帽都是在这个目录下/etc/sysconfig/network-scripts
cp ifcfg-eth0 ifcfg-eth0:1
然后更改其中的ip即可，重启网卡就行。