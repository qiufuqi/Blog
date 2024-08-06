---
title: Cisco交换机常用命令
date: 2023-03-28
tags:
  - 交换机
  - Cisco
categories: 
- 运维
- 交换机
- Cisco
keywords: '交换机,Cisco'
cover: https://qiufuqi.github.io/img/hexo/20230406153459.png
abbrlink: cisco_bash
url: cisco_bash
comments: false
---

https://blog.csdn.net/lzl10211345/article/details/126892519

VLAN（Virtual Local Area Network）的中文名为”虚拟局域网”。 
虚拟局域网（VLAN）是一组逻辑上的设备和用户，这些设备和用户并不受物理位置的限制，可以根据功能、部门及应用等因素将它们组织起来，相互之间的通信就好像它们在同一个网段中一样，由此得名虚拟局域网。VLAN是一种比较新的技术，工作在OSI参考模型的第2层和第3层，一个VLAN就是一个广播域，VLAN之间的通信是通过第3层的路由器来完成的。与传统的局域网技术相比较，VLAN技术更加灵活，它具有以下优点： 网络设备的移动、添加和修改的管理开销减少；可以控制广播活动；可提高网络的安全性。 
在计算机网络中，一个二层网络可以被划分为多个不同的广播域，一个广播域对应了一个特定的用户组，默认情况下这些不同的广播域是相互隔离的。不同的广播域之间想要通信，需要通过一个或多个路由器。这样的一个广播域就称为VLAN。
![](https://qiufuqi.github.io/img/hexo/20230325141235.png)


**VLAN的号码范围：**
1		默认
2-1001		正常
1025-4094	扩展

# 交换机基本命令
## 基本操作
``` bash
Switch# enable              # 进入特许模式
Switch# exit end            # 退出
Switch# write               # 保存
Switch# no *********        # 删除某条命令
Switch# no shutdown         # 启用某个端口
```

https://jingyan.baidu.com/article/6c67b1d6bdce8d2787bb1e99.html
### 交换机端口信息
``` bash
# 查看端口配置信息
Switch# sh run                     [show running-config]
Switch# sh run int f0/1

# 查看端口详细信息
Switch# sh int                     [show interfaces]           # 查看所有端口的详细状态信息
Switch# sh int f0/1                [show interfaces f0/1]      # 查看单一端口的详细状态信息

Switch# sh int status              [show interfaces status]    # 查看端口的简要状态信息
Switch# sh ip int b                [show ip interface brief]  # 查看端口的简要状态信息
Switch# sh int f0/1 status         [show interfaces f0/1 status] # 查看端口的简要信息


Switch# sh access                  [show access-lists] # 查看访问列表配置

# TCP/IP协议相关命令
Switch# show ip access-lists 35    #显示IP访问列表（1-199）
Show ip arp                        #显示路由器的ARP缓存（IP、MAC、封装类型、接口）
Show ip protocols                  #显示运行在路由器上的IP路由协议的信息
Show ip route                      #显示IP路由表中的信息
Show ip traffic                    #显示IP流量统计信息

```
### 三层交换机端口
``` bash
# 路由器或者三层交换机 设置网关 用于通信
Switch(config)# ip routing                                   # 启动三层交换机路由功能
Switch(config)# interface vlan 1                            # 添加设置关联Vlan号为1的路由端口
Switch(config-if)# ip address 192.168.10.1 255.255.255.0    # 为该路由端口设置IP和子网掩码
Switch(config-if)# no ip address 192.168.10.1 255.255.255.0 # 删除为该路由端口设置IP和子网掩码
Switch(config-if)# no shutdown                              # 启动该端口
Switch(config-if)# exit
```
### 路由器子端口
``` bash
# 路由器开启子端口，并设置IP
Router> enable
Router# configure terminal
Router(config)# interface FastEthernet 0/0
Router(config-if)# no shutdown                                  # 启动Fa0/0端口
# 添加设置Fa0/0端口的子端口Fa0/0.1，同理Fa0/1端口的子端口可以为Fa0/1.1，Fa0/0.6等
Router(config-if)# interface FastEthernet 0/0.1                 
# 对该子端口Fa0/0.1进行802.1q协议的封装，后面的数字 1 代表是的侦听VLAN号为 1 的传输数据
Router(config-subif)# encapsulation dot1Q 1
Router(config-subif)# ip address 192.168.10.1 255.255.255.0     # 设置该子端口Fa0/0.1的IP和子网掩码
Router(config-subif)#no shutdown
Router(config-subif)# end                                       # 完成设置退出
Router# show ip route                                           # 查看路由信息
```

## VLAN相关
### 创建VLAN
``` bash
# 进入VLAN数据库，创建VLAN (老型号交换机)
Switch# vlan database 或 vlan data     # 进入vlan列表 
Switch(vlan)# vlan 10 name IT          # 划分VLAN 10，名称为IT 

# 全局配置模式下：(新型号交换机)
Switch#conf t                          # 进入配置模式
Switch(config)#vlan 11                 # 进入vlan 没有则创建
Switch(config-vlan)#name caiwu         # 命名vlan
```
### 查看VLAN
``` bash
Switch# show vlan            # 查看所有vlan信息
Switch# show vlan brief      # 查看摘要
Switch#show vlan id 1        # 只查看vlan 1 的信息
```
### 删除VLAN
``` bash
# 在VLAN数据库模式下
switch# vlan database
Switch(vlan)#no vlan 11

# 全局配置模式下
Switch#conf t
Switch(config)#no vlan 10
```

### 端口加入VLAN
``` bash
# 单个端口加入vlan
Switch#conf t
Switch(config)#int f0/1
Switch(config-if)#switchport access vlan 10

# 多个端口加入vlan
Switch#conf t
Switch(config)#int range f0/1-f0/10
Switch(config-if-range)#switchport access vlan 10

# 还原接口为默认配置
Switch(config)#default int f0/1
```
### 端口删除VLAN
``` bash
# 单个端口退出vlan
Switch#conf t
Switch(config-if)#no switchport access vlan 10

# 多个端口退出vlan
Switch#conf t
Switch(config)#int range f0/1-f0/10
Switch(config-if-range)#no switchport access vlan 10
```

## 接口模式
以太网端口的链路类型有三种：
- Access 类型：端口只能属于1 个VLAN，一般用于交换机与终端用户之间的连接；
- Trunk 类型：端口可以属于多个VLAN，可以接收和发送多个VLAN 的报文，一般用于交换机 之间的连接；
  
**思科所有的trunk链路默认是允许所有vlan通过。**
Trunk标签协议：
ISL:		思科私有标准，仅限于纯思科交换机环境，标签占30字节
802.1q: dot1q   通用标准，任何品牌交换机都可使用，标签只占4字节

### 接口设置trunk
``` bash
Switch#conf t
Switch(config)#int f0/1
Switch(config-if)#switchport mode trunk 
# 或者
Switch(config-if)#sw m t
```
### 查看接口模式
``` bash
Switch#sh int f0/1 sw
Switch#show int f0/1 switchport
```
### 端口不放行vlan
``` bash
switch(config)#int  f0/1
switch(config-if)#switchport trunk allowed vlan remove 10
```
### 端口放行vlan
``` bash
switch(config)#int  f0/1
switch(config-if)#switchport trunk allowed vlan add 10
# 或者
switch(config-if)#switchport trunk allowed vlan 10
# 全部放行
switch(config-if)#switchport trunk allowed vlan all
```






cisco常用交换机命令
# 二层交换机
``` bash
Switch> enable                                  //进入特权模式
Switch# vlan database                           //进入vlan数据库
Switch(vlan)# vlan 2                            //添加一个vlan，vlan号为 2 。默认所有端口处于vlan 1 ，因此本例子只需添加添加 vlan 2
Switch(vlan)# exit                              //退出 vlan数据库
Switch# configure terminal                      //进入全局配置模式
Switch(config)# interface FastEthernet 0/2      //进入端口Fa0/2设置
Switch(config-if)# switchport access vlan 2     //设置端口Fa0/2处于vlan 2，默认所有端口处于vlan 1 ，所以本实例不用对端口Fa0/1设置vlan
Switch(config-if)# exit                         //退出端口Fa0/2设置
Switch(config)# interface FastEthernet 0/3      //进入端口Fa0/3设置
Switch(config-if)# switchport access vlan 2     //设置端口Fa0/3处于vlan 2
Switch(config-if)# end                          //设置完成退出


Switch(config-if)# no shutdown                  //启用某个端口
Switch(config-if)# ip address 192.168.20.1 255.255.255.0          //设置端口IP地址


Switch> enable
Switch# configure terminal
Switch(config)# interface FastEthernet 0/24             //Fa0/24端口连接路由器
Switch(config-if)# switchport mode trunk                //设置该端口vlan模式为trunk
Switch(config-if)# switchport trunk allowed vlan all    //设置该端口trunk模式下接收所有vlan线路的信息
Switch(config-if)# end                                  //完成设置退出
```

``` bash
Router> enable
Router# configure terminal
Router(config)# interface FastEthernet 0/0
Router(config-if)# no shutdown                                  //启动Fa0/0端口
 
Router(config-if)# interface FastEthernet 0/0.1                 //添加设置Fa0/0端口的子端口Fa0/0.1
                                                                //同理Fa0/1端口的子端口可以为Fa0/1.1，Fa0/0.6等
 
Router(config-subif)# encapsulation dot1Q 1                     //对该子端口Fa0/0.1进行802.1q协议的封装
                                                                //后面的数字 1 代表是的侦听VLAN号为 1 的传输数据
 
Router(config-subif)# ip address 192.168.10.1 255.255.255.0     //设置该子端口Fa0/0.1的IP和子网掩码
Router(config-subif)#no shutdown                                //启动该子端口
 
Router(config-if)# interface FastEthernet 0/0.2                 //添加设置Fa0/0端口的子端口Fa0/0.2
Router(config-subif)# encapsulation dot1Q 2                     //对该子端口Fa0/0.2进行802.1q协议的封装
 
Router(config-subif)# ip address 192.168.20.1 255.255.255.0     //设置该子端口Fa0/0.2的IP和子网掩码
Router(config-subif)# no shutdown                               //启动该子端口
 
Router(config-subif)# end                                       //完成设置退出
Router# show ip route                                           //查看路由信息
 
Codes: C - connected, S - static, I - IGRP, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       * - candidate default, U - per-user static route, o - ODR
       P - periodic downloaded static route
 
Gateway of last resort is not set
 
C    192.168.10.0/24 is directly connected, FastEthernet0/0.1
C    192.168.20.0/24 is directly connected, FastEthernet0/0.2   
```
# 三层交换机
普通交换机没有路由功能，而三层交换机可以看作普通交换机和路由器合二为一的机器。 
因此**三层交换机具有路由功能，而且三层交换机的端口是vlan端口**。

主机——三层交换机——主机
![](https://qiufuqi.github.io/img/hexo/20230325141347.png)

三层交换机增加vlan和对主机的vlan划分部分
创建vlan 并 将端口划分至指定vlan下 （参考上面）；启用三层交换机的路由功能
ip routing
``` bash
Switch> enable
Switch# configure terminal
 
Switch(config)#ip routing                                   //启动三层交换机路由功能
 
Switch(config)# interface Vlan 1                            //添加设置关联Vlan号为1的路由端口
Switch(config-if)# ip address 192.168.10.1 255.255.255.0    //为该路由端口设置IP和子网掩码
Switch(config-if)# no shutdown                              //启动该端口
Switch(config-if)# exit                                     //退出该端口
 
Switch(config)# interface Vlan 2                            //添加设置关联Vlan号为2的路由端口
Switch(config-if)# ip address 192.168.20.1 255.255.255.0    //为该路由端口设置IP和子网掩码
Switch(config-if)# no shutdown                              //启动该端口
Switch(config-if)# exit                                     //退出该端口
 
Switch(config)# interface Vlan 3                            //添加设置关联Vlan号为3的路由端口
Switch(config-if)# ip address 192.168.30.1 255.255.255.0    //为该路由端口设置IP和子网掩码
Switch(config-if)# no shutdown                              //启动该端口
Switch(config-if)# end                                      //完成退出
 
Switch# show ip route                                       //查看路由信息
 
Codes: C - connected, S - static, I - IGRP, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       * - candidate default, U - per-user static route, o - ODR
       P - periodic downloaded static route
 
Gateway of last resort is not set
 
C    192.168.10.0/24 is directly connected, Vlan1
C    192.168.20.0/24 is directly connected, Vlan2
C    192.168.30.0/24 is directly connected, Vlan3
```

主机——交换机——三层交换机——交换机——主机
普通交换机添加vlan和对主机vlan划分部分省略。 
若普通交换机连接三层交换机的端口已设置vlan模式为trunk，连接三层交换机后，三层交换机连接普通交换机的端口其模式会自动设置为trunk。
![](https://qiufuqi.github.io/img/hexo/20230325144531.png)

二层交换机设置
``` bash
Switch> enable
Switch# configure terminal
Switch(config)# interface FastEthernet 0/24
Switch(config-if)# switchport mode trunk        //设置Fa0/24端口vlan模式为trunk
Switch(config-if)#end
```
三层交换机设置 和前面类似
``` bash
Switch> enable
Switch#vlan database
Switch(vlan)# vlan 2                                        //添加vlan 2
Switch(vlan)# exit
 
Switch# configure terminal
 
Switch(config)#ip routing                                   //启动三层交换机路由功能
 
Switch(config)# interface Vlan 1                            //添加设置关联Vlan号为1的路由端口
Switch(config-if)# ip address 192.168.10.1 255.255.255.0    //为该路由端口设置IP和子网掩码
Switch(config-if)# no shutdown                              //启动该端口
Switch(config-if)# exit                                     //退出该端口
 
Switch(config)# interface Vlan 2                            //添加设置关联Vlan号为2的路由端口
Switch(config-if)# ip address 192.168.20.1 255.255.255.0    //为该路由端口设置IP和子网掩码
Switch(config-if)# no shutdown                              //启动该端口
Switch(config-if)# end                                      //退出该端口
 
Switch# show ip route                                       //查看路由信息
 
Codes: C - connected, S - static, I - IGRP, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       * - candidate default, U - per-user static route, o - ODR
       P - periodic downloaded static route
 
Gateway of last resort is not set
 
C    192.168.10.0/24 is directly connected, Vlan1
C    192.168.20.0/24 is directly connected, Vlan2
```
