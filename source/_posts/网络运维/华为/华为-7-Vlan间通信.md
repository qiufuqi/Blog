---
title: 华为-7-Vlan间通信
date: 2024-2-12
tags:
  - 华为
  - VLAN
  - 通信
categories: 
- 运维
- 华为
- VLAN间通信
keywords: '华为,VLAN间通信'
cover: https://qiufuqi.github.io/img/hexo/20230327084004.png
abbrlink: huawei_vlan_c
url: huawei_vlan_c
comments: false
---
实现两个VLAN之间的通信，需要三层路由功能的介入，有两种方式可以实现这个目标。一个是在网络中添加路由器，路由器作为三层设备为VLAN之间的流量执行路由，第二种是在交换机上启用VLANIF接口，使用这个虚拟接口执行三层路由功能（部分交换机才支持）。

# 添加路由器
添加路由器，为VLAN之间的执行路由。
交换机上的每个VLAN都要与路由器上的三层接口对应，实际应用中，我们使用一个路由器的接口就可以实现多个VLAN之间的路由转发：通过**Dot1q终结子端口**以实现不同VLAN之间的通信。此时交换机和路由器之间只有一条链路。
每个子接口都是相应VLAN的终结，即子接口作为VLAN中终端设备的默认网关。交换机与路由器相连端口需要传输两个VLAN的流量，因此将其配置为trunk，并放行相应的VLAN流量。
对于路由器接口，需要进行如下配置:
- 创建子端口
- 配置相应的VLAN的Dot1q终结
- 配置IP地址，作为终端设备的默认网关

实验拓扑图如下：
![](https://qiufuqi.github.io/img/hexo/20240212155102.png)
## 配置交换机
配置交换机SW1，和终端设备相连端口设置为access，并加入相应的vlan，和路由器R1相连的端口设置为trunk口并放行相应的vlan。
``` bash
[SW1]vlan batch 10 20
[SW1]int e0/0/1
[SW1-Ethernet0/0/1]port link-type access 
[SW1-Ethernet0/0/1]port default vlan 10
[SW1]int e0/0/2
[SW1-Ethernet0/0/2]port link-type access 
[SW1-Ethernet0/0/2]port default vlan 20

[SW1]int g0/0/1
[SW1-GigabitEthernet0/0/1]port link-type trunk 
[SW1-GigabitEthernet0/0/1]port trunk allow-pass vlan 10 20
```
## 配置子接口
在路由器上配置Dot1q终结子接口
- **interface interface-type interface-number.subinterface-number**：系统视图命令，创建子接口并进入子接口配置视图
- **ip address ip-address mask-length**：子接口视图命令，配置子接口IP地址，并将**子接口的IP地址作为相应VLAN的默认网关**。
- dot1q termination vid low-pe-vid：子接口视图命令，指定该子接口终结的VLAN；
  - 每个子接口只能关联并终结一个VLAN；
  - 同一个主接口下的不同子接口不能关联相同的VLAN；
  - 不同主接口下的子接口可以关联相同的VLAN
- arp broadcast enable：子接口视图命令，在该子接口上启用ARP广播功能。

```bash
[R1]int g0/0/1.10
[R1-GigabitEthernet0/0/1.10]ip address 172.16.10.1 24
[R1-GigabitEthernet0/0/1.10]dot1q termination vid 10
[R1-GigabitEthernet0/0/1.10]arp broadcast enable
[R1]int g0/0/1.20
[R1-GigabitEthernet0/0/1.20]ip address 172.16.20.1 24
[R1-GigabitEthernet0/0/1.20]dot1q termination vid 20
[R1-GigabitEthernet0/0/1.20]arp broadcast enable
```
子端口的IP地址作为相应vlan的网关，此时PC1和PC2能够进行互通。
使用**display arp查看路由器上**MAC地址与VLAN的关系
使用**display mac-address查看交换机上**MAC地址与VLAN的关系
```bash
[R1]dis arp
IP ADDRESS      MAC ADDRESS     EXPIRE(M) TYPE        INTERFACE   VPN-INSTANCE 
                                          VLAN/CEVLAN PVC                      
------------------------------------------------------------------------------
172.16.10.1     00e0-fc8e-402b            I -         GE0/0/1.10
172.16.10.10    5489-9804-0690  19        D-0         GE0/0/1.10
                                            10/-
172.16.20.1     00e0-fc8e-402b            I -         GE0/0/1.20
172.16.20.20    5489-98f9-72fb  19        D-0         GE0/0/1.20
                                            20/-
------------------------------------------------------------------------------
Total:4         Dynamic:2       Static:0     Interface:2    
```
# 配置VLANIF接口
在实际应用中，每个VLAN中的终端设备数量庞大，启用子接口的arp广播功能可能会因大量arp广播请求而造成链路资源浪费，影响网关设备的运行。在使用VLANIF接口实现VLAN间通信中，如果连接终端的设备具有三层路由功能，则无须额外的路由器。
配置vlanif命令如下：
- **interface vlanif vlan-id**：系统视图命令，创建vlanif接口并进入vlanif接口视图
- **ip address ip-address mask-length**：接口视图命令，配置vlanif接口的ip地址，即vlan中终端设备的网关

实验拓扑图如下：
![](https://qiufuqi.github.io/img/hexo/20240212160308.png)
``` bash
[SW1]interface Vlanif 10
[SW1-Vlanif10]ip address 172.16.10.1 24
[SW1]interface Vlanif 20
[SW1-Vlanif20]ip address 172.16.20.1 24
```
vlanif接口时逻辑接口，因此创建后它的状态就是UP，配置了IP后，它的线路协议也会进入UP，此时PC1和PC2能够进行互通。
交换机启用了三层功能，使用**display arp查看交换机上**学习到的MAC地址。
使用**display interface vlanif vlan-id**查看vlanif接口的信息
```bash
[SW1]dis arp
IP ADDRESS      MAC ADDRESS     EXPIRE(M) TYPE INTERFACE      VPN-INSTANCE      
                                          VLAN 
------------------------------------------------------------------------------
172.16.10.1     4c1f-cc97-753e            I -  Vlanif10
172.16.10.10    5489-9804-0690  18        D-0  Eth0/0/1
                                          10
172.16.20.1     4c1f-cc97-753e            I -  Vlanif20
172.16.20.20    5489-98f9-72fb  18        D-0  Eth0/0/2
                                          20
------------------------------------------------------------------------------
Total:4         Dynamic:2       Static:0     Interface:2  

[SW1]display interface Vlanif 10
Vlanif10 current state : UP
Line protocol current state : UP
Last line protocol up time : 2024-02-12 15:56:00 UTC-08:00
Description:
Route Port,The Maximum Transmit Unit is 1500
Internet Address is 172.16.10.1/24
IP Sending Frames' Format is PKTFMT_ETHNT_2, Hardware address is 4c1f-cc97-753e
Current system time: 2024-02-12 16:01:21-08:00
    Input bandwidth utilization  : --
    Output bandwidth utilization : --
```