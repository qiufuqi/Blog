---
title: 华为-13-ARP协议
date: 2024-2-19
tags:
  - 华为
  - ARP
categories: 
- 运维
- 华为
- ARP
keywords: '华为,ARP'
cover: https://qiufuqi.github.io/img/hexo/20230327084004.png
abbrlink: huawei_arp
url: huawei_arp
comments: false
---
[文章参考华为官网](https://support.huawei.com/enterprise/zh/doc/EDOC1000069491/67fd8bfb#ZH-CN_TASK_0177091436)
# ARP介绍
地址解析协议，即ARP（Address Resolution Protocol），是根据IP地址获取物理地址的一个TCP/IP协议。主机发送信息时将包含目标IP地址的ARP请求广播到局域网络上的所有主机，并接收返回消息，以此确定目标的物理地址；收到返回消息后将该IP地址和物理地址存入本机ARP缓存中并保留一定时间，下次请求时直接查询ARP缓存以节约资源。

# 静态ARP
**静态ARP表项是指网络管理员手工建立IP地址和MAC地址之间固定的映射关系**
正常情况下网络中设备可以通过ARP协议进行ARP表项的动态学习，生成的动态ARP表项可以被老化，可以被更新。但是当网络中存在ARP攻击时，设备中动态ARP表项可能会被更新成错误的ARP表项，或者被老化，造成合法用户通信异常。**静态ARP表项不会被老化，也不会被动态ARP表项覆盖**，可以保证网络通信的安全性。
静态ARP表项可以**限制本端设备和指定IP地址的对端设备通信时只使用指定的MAC地址**，此时攻击报文无法修改本端设备的ARP表中IP地址和MAC地址的映射关系，从而保护了本端设备和对端设备间的正常通信。一般在网关设备上配置静态ARP表项。
设备上配置的静态ARP表项数目不能大于设备静态ARP表项规格，可以执行命令**display arp statistics all**查看设备上已有的ARP表项数目。

**portswitch**命令用来配置将以太网接口从三层模式切换到二层模式。
**undo portswitch**命令用来配置将以太网接口从二层模式切换到三层模式。

## 实验参考
企业通过Switch实现各个部门之间的互连，且各个部门加入不同的VLAN。总裁办公室和文件备份服务器采取手工方式分配已经获取到固定IP地址，市场部和研发部主机通过DHCP方式已经获取到动态IP地址。由于市场部拥有访问外网的权利，主机经常会感染ARP病毒，攻击Switch并修改Switch上的动态ARP表项，造成总裁办公室与外界的通信中断以及各个部门不能正常访问文件备份服务器。公司希望在Switch上配置静态ARP表项，以保证总裁办公室与外界的通信安全，并保证各个部门能正常访问文件备份服务器。
![](https://qiufuqi.github.io/img/hexo/20240219223226.png)

静态ARP的配置思路如下：
1. 在Switch上为总裁办公室主机配置静态ARP表项，防止总裁办公室主机的ARP表项被ARP攻击报文修改，造成总裁办公室与外界的通信中断。
2. 在Switch上为文件备份服务器配置静态ARP表项，防止文件备份服务器的ARP表项被ARP攻击报文修改，造成各个部门不能正常访问文件备份服务器。

``` bash
# 为指定主机配置静态ARP表项
[Switch] arp static 10.10.10.1 00e0-fc01-0001 vid 10 interface gigabitethernet 1/0/1 

# 为重要服务器如文件备份服务器配置静态ARP表项
[Switch] arp static 10.164.10.1 00e0-fc02-1234 interface gigabitethernet 1/0/2 
[SW]display arp static
IP ADDRESS      MAC ADDRESS     EXPIRE(M) TYPE INTERFACE      VPN-INSTANCE      
                                          VLAN 
------------------------------------------------------------------------------
10.10.10.1      00e0-fc01-0001            S--  GE0/0/1
                                          10
------------------------------------------------------------------------------
Total:1         Dynamic:0       Static:1     Interface:0
```
SW配置如下
```bash
#
sysname Switch
#
vlan batch 10
#
interface Vlanif10
 ip address 10.164.1.20 255.255.255.0
#
interface GigabitEthernet1/0/1
 port link-type access
 port default vlan 10
#
interface GigabitEthernet1/0/2
 undo portswitch
 ip address 10.164.10.10 255.255.255.0
#
interface GigabitEthernet1/0/3
 undo portswitch
 ip address 10.164.20.1 255.255.255.0
#
arp static 10.164.1.1 00e0-fc01-0001 vid 10 interface GigabitEthernet1/0/1
arp static 10.164.10.1 00e0-fc02-1234 interface GigabitEthernet1/0/2
#
return
```

# 路由式Proxy ARP
企业内部进行子网划分时，可能会出现两个子网网络属于同一网段，但是却不属于同一物理网络的情况，两个子网网络间被交换机分隔（比如不同城市的分公司，使用同一个网段，却不属于同一个物理网络，要求实现互相通信）。这时可以通过修改网络内主机的路由信息，使发往其它子网的数据先发送到连接不同子网的网关设备上，再由网关设备转发此数据报文。但是这种解决方案需要配置子网中所有主机的路由，并不便于管理和维护。

在网关上部署路由式Proxy ARP功能，可以有效解决子网划分带来的管理和维护方面的问题。路由式Proxy ARP可以使IP地址属于同一网段却不属于同一物理网络的主机间能够相互通信，并且主机上不需要配置缺省网关，便于管理和维护。
在设备上使能路由式Proxy ARP后，与设备相连的主机上应该减小ARP表项老化超时时间，使无效的ARP表项尽快失效，减少发给交换机而交换机不能转发的报文

## 实验参考
某企业的子公司A和子公司B位于不同城市，且所使用的IP地址部署为同一个网段（172.16.0.0/16）。连接子公司A的Switch_1与连接子公司B的Switch_2之间路由可达。由于两个子公司之间被设备间隔，属于不同的广播域，因此无法在同一个局域网内实现互通；子公司的主机没有配置默认网关，无法实现跨网段互通。现在公司需要在不改变主机配置的情况下，实现两个子公司之间的通信。
![](https://qiufuqi.github.io/img/hexo/20240219222858.png)

采用如下的配置思路实现子公司A和子公司B之间互通：
1. 在Switch_1上将连接子公司A的接口划分到VLAN10，在Switch_2上将连接子公司B的接口划分到VLAN20。
2. 在Switch_1和Switch_2的VLANIF接口上使能路由式Proxy ARP功能，实现子公司A和子公司B互通。

``` bash
# 配置路由式Proxy ARP
[Switch_1-Vlanif10] arp-proxy enable
[Switch_2-Vlanif20] arp-proxy enable

# 查询IP对应的mac地址情况
[Switch_1] display arp interface vlanif 10
IP ADDRESS      MAC ADDRESS     EXPIRE(M) TYPE        INTERFACE   VPN-INSTANCE                                                      
                                          VLAN/CEVLAN                                                                               
------------------------------------------------------------------------------                                                      
172.16.1.1      00e0-fc12-3456            I -         Vlanif10                                                                     
------------------------------------------------------------------------------                                                      
Total:1         Dynamic:0       Static:0     Interface:1 
```
SW的配置文件如下
```bash
# SW1
sysname Switch_1
#
vlan batch 10
#
interface Vlanif10
 ip address 172.16.1.1 255.255.255.0
 arp-proxy enable
#
interface GigabitEthernet1/0/1
 port link-type access
 port default vlan 10
#
return

# SW2
sysname Switch_2
#
vlan batch 20
#
interface Vlanif20
 ip address 172.16.2.1 255.255.255.0
 arp-proxy enable
#
interface GigabitEthernet1/0/1
 port link-type access
 port default vlan 20
#
return
```