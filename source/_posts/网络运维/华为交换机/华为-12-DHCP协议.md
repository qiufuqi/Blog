---
title: 华为-12-DHCP协议
date: 2024-2-19
tags:
  - 华为
  - DHCP
categories: 
- 运维
- 华为
- DHCP
keywords: '华为,DHCP'
cover: https://qiufuqi.github.io/img/hexo/20230327084004.png
abbrlink: huawei_dhcp
url: huawei_dhcp
comments: false
---

[文章参考华为官网](https://support.huawei.com/enterprise/zh/doc/EDOC1000069491/6946084b)
# DHCP服务器简介
动态主机配置协议DHCP（Dynamic Host Configuration Protocol）采用客户端/服务器模式对用户的网络参数进行动态配置和集中管理。其中，DHCP服务器通过地址池为用户分配IP地址等网络参数。地址池分为**接口地址池和全局地址池**两种。
# 接口地址池配置
接口地址池配置方式简单，只能用于用户与DHCP服务器在同一个网段的场景，并且只能给对应接口下的用户分配IP地址等网络参数；适用于设备数量有限、配置以及维护量可控的小型网络。在用户网关设备上配置基于接口地址池的DHCP服务器功能之后，对应接口下的固定主机、移动终端等都可以自动获取IP地址等网络参数，不需要用户手动配置修改。

``` bash
# 使能DHCP服务，缺省未使能
[Switch] dhcp enable

[Switch] interface vlanif 10
[Switch-Vlanif10] ip address 10.1.1.1 24  //企业为固定办公终端的网段

[Switch-Vlanif10] dhcp select interface  //使能接口采用接口地址池的DHCP服务器功能，缺省未使能
[Switch-Vlanif10] dhcp server lease day 30  //租期的缺省值为1天，修改租期为30天
[Switch-Vlanif10] dhcp server dns-list 8.8.8.8
[Switch-Vlanif10] dhcp server static-bind ip-address 10.1.1.100 mac-address 00e0-fc12-3456  //为Client_1分配固定的IP地址
[Switch-Vlanif10] quit

# 配置DHCP数据保存功能，设备发生故障时，可以在系统重启后，执行命令dhcp server database recover，从存储设备文件恢复DHCP数据。
[Switch] dhcp server database enable

# 查看vlanif10的IP分配情况
[Switch] display ip pool interface vlanif10
```

# 全局地址池配置
与接口地址池相比，全局地址池可应用于大型网络，推荐在核心层设备上配置基于全局地址池的DHCP服务器功能或在服务器区域搭建一台专门的DHCP服务器统一分配IP地址等网络参数，而用户网关设备上只需要启用简单的DHCP中继功能即可
``` bash
# 使能DHCP服务，缺省未使能
[Switch] dhcp enable

# 创建pool地址池
[Switch] ip pool pool1
[Switch-ip-pool-pool1] network 10.1.1.0 mask 255.255.255.128
[Switch-ip-pool-pool1] dns-list 10.1.2.3
[Switch-ip-pool-pool1] gateway-list 10.1.1.1
[Switch-ip-pool-pool1] lease day 10
[Switch-ip-pool-pool1] quit

# 端口下使能DHCP
[Switch] interface vlanif 10
[Switch-Vlanif10] dhcp select global
[Switch-Vlanif10] quit

# 查看全局地址池pool1的分配情况
display ip pool name pool1
```
# DHCP中继
DHCP中继用于在DHCP服务器和客户端之间转发DHCP报文。当**DHCP服务器与客户端不在同一个网段时，需要配置DHCP中继**。对于DHCP客户端来说，DHCP中继就是DHCP服务器；对于DHCP服务器来说，DHCP中继就是DHCP客户端。

DHCP中继适用于用户网关设备众多且分布零散的大型网络。为减少维护工作量，网络管理员不想在每个汇聚层交换机（用户网关）上都配置DHCP服务器功能，而希望在核心层设备上配置DHCP服务器功能或在服务器区域部署一台专门的DHCP服务器。此时，作为用户网关的汇聚层交换机上就需要配置DHCP中继功能，实现DHCP服务器与客户端之间的DHCP报文交互。

某企业将DHCP服务器部署在核心交换机上，DHCP服务器与企业内的终端不在同一个网段。企业希望使用该DHCP服务器为终端动态分配IP地址。
![](https://qiufuqi.github.io/img/hexo/20240220195119.png)
## 配置思路
设备作为DHCP中继的配置思路如下：
1. 在汇聚层交换机SwitchA（用户网关）上配置DHCP中继，实现设备作为DHCP中继转发终端与DHCP服务器之间的DHCP报文。
2. 在核心层交换机SwitchB上，配置基于全局地址池的DHCP服务器，实现DHCP服务器从全局地址池中选择IP地址分配给企业终端。

在汇聚成交换机上配置DHCP中继
``` bash
[SwitchA] dhcp enable  //使能DHCP服务，缺省未使能
[SwitchA] interface vlanif 100
[SwitchA-Vlanif100] ip address 10.10.20.1 24
[SwitchA-Vlanif100] dhcp select relay   //使能DHCP中继功能，缺省未使能
[SwitchA-Vlanif100] dhcp relay server-ip 192.168.20.2  //配置DHCP中继代理的DHCP服务器的IP地址
[SwitchA-Vlanif100] quit
```
在核心交换机上配置全局地址池
``` bash
[SwitchB] interface vlanif 200 
[SwitchB-Vlanif200] ip address 192.168.20.2 24
[SwitchB-Vlanif200] dhcp select global  //使能接口采用全局地址池的DHCP服务器功能，缺省未使能

[SwitchB] ip pool pool1
[SwitchB-ip-pool-pool1] network 10.10.20.0 mask 24  //配置全局地址池的网段和掩码
[SwitchB-ip-pool-pool1] gateway-list 10.10.20.1  //配置为终端分配的网关地址
[SwitchB-ip-pool-pool1] quit
```