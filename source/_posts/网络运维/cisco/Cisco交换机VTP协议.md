---
title: Cisco交换机VTP协议
date: 2022-11-24
tags:
  - 交换机
  - Cisco
  - VTP
categories: 
- 运维
- 交换机
- Cisco
keywords: '交换机,Cisco,VTP'
cover: https://qiufuqi.github.io/img/hexo/20230406153459.png
abbrlink: cisco_vtp
url: cisco_vtp
comments: false
---

https://blog.csdn.net/qq_60503432/article/details/127194455

VTP（VLAN Trunk Protocol）---CISCO私有的协议
VTP一个可以说很方便的协议，学习Cisco时也很常用的协议，他的作用就是可以在有大量交换设备，需要配置类似的vlan划分的时候，简化配置过程，使各个交换机互相学习Vlan Database

为什么要使用VTP呢 ?
考虑到在一个三层模型拥有多台交换设备的网络环境中，为了使得在每台交换设备上VLAN信息一致，那必须得在每台设备上单独配置相同的VLAN。如果设备足够多那么这样的效率是很低的并且如果经常需要临时的创建和删除某些VLAN在这样的大型网络中效率更低，VTP允许在一台交换机上添加、删除、修改VLAN信息然后同步到其他的交换机。
# VTP模式
![](https://qiufuqi.github.io/img/hexo/20230406162245.png)
在VTP协议中我们把设备大体分为两类：Server和Client
Server端能够添加、删除、修改vlan信息并且把vlan信息保存在flash:vlan.dat中
Client只能向Server同步vlan信息并且vlan信息无法保存，每次重启都要向Server同步

server        创建、修改、删除、发送或转发通告（发送自己的消息，转发别人的消息）、同步（接受别人发来的消息）、保存
client        转发通告、同步、不保存
transparent   创建、修改、删除、转发通告、不同步、保存

为了丰富VTP的功能除了Server和Client模式之外，还设置了一个Transparent模式；
Transparent会传递vlan信息，但自己本身不同步Server的vlan信息，vlan信息保存在nvram:startup-config中；
Transparent对vlan的操作只在本交换机上有效；
三种模式对设备的数目没有限制（在一个交换网络中没有规定只能有一台server、client、transparent）

# VTP运行
![](https://qiufuqi.github.io/img/hexo/20230406163357.png)

VTP帧发向组播MAC地址：0100.0CCC.CCCC
VTP同步的关键是配置修订号（指的是对设备进行了多少次的更新、修改）
VTPserver和clinent同步到最新的配置版本号，VTP通告在变化时，每5分钟发送。
配置修订号越大设备认为配置越新，**修订号大的同步vlan信息给修订号小的，修订号小的设备vlan信息被覆盖**；
正是因为同步是依据配置修订号来，导致VTP很少有人用。

局限性：
1. 依据修订号来同步vlan信息
2. 思科私有

# VTP作用
VTP维护整个管理域VLAN信息的一致性
VTP仅在Trunk端口上发送通告
同步到最新的VLAN信息

# VTP配置
## 配置需求
Trunk接口，VTP域名，指定VTP模式（默认server），配置VTP密码
配置命令
``` bash
Switch(config)#vtp domainTEST
Changing VTP domain name from NULL to TESTS
witch(config)#vtp mode server
Switch(config)#vtp password cisco
Setting device VLAN database password to cisco
```
验证VTP配置


































