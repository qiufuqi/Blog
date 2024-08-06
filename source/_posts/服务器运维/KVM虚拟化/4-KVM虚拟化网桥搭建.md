---
title: KVM虚拟化网桥搭建
date: 2022-08-16 16:19:45
tags:
  - KVM
  - Birdge
categories: 
- 运维
- 虚拟化
- 网桥
keywords: 'Linux,CentOS,KVM,网桥'
description: KVM虚拟化网桥搭建
cover: https://qiufuqi.github.io/img/hexo/20231205140922.png
abbrlink: kvm_bridge
comments: false
top: 86
---

Linux环境部署KVM虚拟化平台，系统版本centos7.6，GUI桌面以及虚拟化支持（安装时）。

## Bridge基本原理
一般来说，KVM客户机有两种网络连接方式：NAT与Bridge。
NAT方式：让虚拟机访问主机、互联网或本地网络上的资源的简单方法，但是不能从网络或其他的客户机访问客户机，性能上也需要大的调整。
Virtual Bridge：这种方式要比用户网络复杂一些，但是设置好后客户机与互联网，客户机与主机之间的通信都很容易。客户机和子网里面的机器能够互相通信。可以使虚拟机成为网络中具有独立IP的主机。
桥接网络（也叫物理设备共享）被用作把一个物理设备复制到一台虚拟机。

网络虚拟化是虚拟化技术中最复杂的部分，也是非常重要的资源。


## 单Vlan Bridge创建步骤
在eno3上搭建一座Bridge=breno3，此桥的所有流量均从eno3发送，KVM虚拟机网络挂载到breno3上即可进行网络访问，breno3所在vlan和eno3所在vlan一致。
原理图:
![](https://qiufuqi.github.io/img/hexo/20231205142943.png)

试验使用ifcfg-eno3网口, 进入网络配置目录：cd /etc/sysconfig/network-scripts
文件可直接创建 或者使用命令创建
``` bash
1 创建网桥
[root@node1 network-scripts]# brctl addbr breno3
2 将br0与你的物理网卡进行绑定
[root@node1 network-scripts]# brctl addif breno3 eno3
3 如果要打开STP协议：
[root@node1 network-scripts]# brctl stp breno3 on


# brctl 其他命令
brctl delif breno3 eno3    #解除绑定
ifconfig breno3 down     #关闭breno3,不关闭删不掉
brctl delbr breno3       #删除breno3

#brctl show 查看网桥连接
addbr 添加网桥
delbr 删除网桥
addif 添加接口
delif 删除接口

```

### 修改物理端口
``` bash
[root@node1 network-scripts]# vi ifcfg-eno3
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=eno3
UUID=2f5fa659-68ca-4bee-86c9-2d9d51a4f681
DEVICE=eno3
ONBOOT=yes
#IPADDR=10.128.1.10
#PREFIX=24
#GATEWAY=10.128.1.1
#DNS1=192.168.0.168
IPV6_PRIVACY=no

#在eno3上搭建一座Bridge,此桥的所有流量均从eno3发送
BRIDGE=breno3
```
### 添加对应Birdge
``` bash
[root@node1 network-scripts]# vi ifcfg-breno3 
TYPE=Bridge
DEVICE=breno3
ONBOOT=yes
BOOTPROTO=static
NAME=breno3
IPADDR=10.128.1.10
PREFIX=24
GATEWAY=10.128.1.1
DNS1=192.168.0.168
```
### 重启网络，查看
经过检测，KVM虚拟机可正常访问网络
``` bash
service network restart

#使用brctl show查看  vnet0为kvm虚拟机网卡
[root@node1 network-scripts]# ifconfig
[root@node1 network-scripts]# brctl show
bridge name	bridge id		STP enabled	interfaces
breno3		8000.6cae8b406513	no		eno3
							vnet0

```

## 多Vlan Bridge创建步骤
一个Vlan为一个网段，同一台机器上的虚拟机可能归属不同的vlan，此时需要在宿主机创建多个vlan，将各个虚拟机划分到不同的vlan下，虚机产生的流量将带上不同vlan标识。
注意：eno5在交换机端更改为trunk口，宿主机用软件实现了一个交换机
原理图:
![](https://qiufuqi.github.io/img/hexo/20231205142721.png)


### 创建端口vlan
创建vlan10 同理可创建vlan12
``` bash
[root@node1 network-scripts]# vi ifcfg-eno5.10
DEVICE="eno5.10"
VLAN="yes"
ONBOOT="yes"
BOOTPROTO="none"
BRIDGE=brvlan-10
```
### 创建brvlan-10
``` bash
[root@node1 network-scripts]# cat ifcfg-brvlan-10
TYPE=bridge
BOOTPROTO=static
NAME=brvlan-10
DEVICE=brvlan-10
ONBOOT=yes
```
### 重启网络，查看
经过检测，KVM虚拟机挂载到对应vlan下，可正常访问网络
``` bash
service network restart
```

## 网卡绑定bond
网卡bond（绑定），也称作网卡捆绑。就是将两个或者更多的物理网卡绑定成一个虚拟网卡。
网卡绑定的目的：
1.提高网卡的吞吐量。
2.增强网络的高可用，同时也能实现负载均衡。

网卡配置 bond （绑定）bond模式：
（1）Mode=0(balance-rr) 表示负载分担round-robin，平衡轮询策略，具有负载平衡和容错功能bond的网卡MAC为当前活动的网卡的MAC地址，需要交换机设置聚合模式，将多个网卡绑定为一条链路。
（2）Mode=1(active-backup) 表示主备模式，具有容错功能，只有一块网卡是active,另外一块是备的standby，这时如果交换机配的是捆绑，将不能正常工作，因为交换机往两块网卡发包，有一半包是丢弃的。
（3）Mode=2(balance-xor) 表示XOR Hash负载分担（异或平衡策略），具有负载平衡和容错功能每个slave接口传输每个数据包和交换机的聚合强制不协商方式配合。（需要xmit_hash_policy）。
（4）Mode=3(broadcast) 表示所有包从所有interface发出，广播策略，具有容错能力，这个不均衡，只有冗余机制…和交换机的聚合强制不协商方式配合。
（5）Mode=4(802.3ad) 表示支持802.3ad协议（IEEE802.3ad 动态链接聚合） 和交换机的聚合LACP方式配合（需要xmit_hash_policy）。
（6）Mode=5(balance-tlb) 适配器传输负载均衡，并行发送，无法并行接收，解决了数据发送的瓶颈。 是根据每个slave的负载情况选择slave进行发送，接收时使用当前轮到的slave。
（7）Mode=6(balance-alb) 在5的tlb基础上增加了rlb。适配器负载均衡模式并行发送，并行接收数据包。

5和6不需要交换机端的设置，网卡能自动聚合。4需要支持802.3ad。0，2和3理论上需要静态聚合方式，但实测中0可以通过mac地址欺骗的方式在交换机不设置的情况下不太均衡地进行接收。

常用的有三种
mode=0：平衡负载模式，有自动备援，但需要”Switch”支援及设定。
mode=1：自动备援模式，其中一条线若断线，其他线路将会自动备援。
mode=6：平衡负载模式，有自动备援，不必”Switch”支援及设定。

原理图：
![](https://qiufuqi.github.io/img/hexo/20231205143019.png)

### 更改端口配置
首先更改eno4配置,绑定到bond6; 同理,其他端口可同样操作（对应交换机为trunk,否则vlan之间无法通信）
``` bash
[root@node1 network-scripts]# vi ifcfg-eno4
TYPE=Ethernet
BOOTPROTO=none
NAME=eno4
DEVICE=eno4
ONBOOT=yes
MASTER=bond6
SLAVE=YES
[root@node1 network-scripts]# vi ifcfg-eno5
TYPE=Ethernet
BOOTPROTO=none
NAME=eno5
DEVICE=eno5
ONBOOT=yes
MASTER=bond6
SLAVE=YES
```

### 创建master=bond0
网卡绑定模式
``` bash
[root@node1 network-scripts]# vi ifcfg-bond6 
DEVICE=bond6
TYPE=Bond
NAME=bond6
BONDING_MASTER=yes
BOOTPROTO=static
USERCTL=no
ONBOOT=yes
BONDING_OPTS="mode=6 miimon=100"
BRIDGE=br6
```
### 搭建桥br6
``` bash
[root@node1 network-scripts]# cat ifcfg-br6
TYPE=Bridge
DEVICE=br6
ONBOOT=yes
BOOTPROTO=static
NAME=br6
```

### 查看bond状态
可查看当前所使用端口
``` bash
[root@node1 network-scripts]# cat /proc/net/bonding/bond6 
Ethernet Channel Bonding Driver: v3.7.1 (April 27, 2011)

Bonding Mode: adaptive load balancing
Primary Slave: None
Currently Active Slave: eno4
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0

Slave Interface: eno5
MII Status: up
Speed: 1000 Mbps
Duplex: full
Link Failure Count: 1
Permanent HW addr: 6c:ae:8b:40:65:15
Slave queue ID: 0

Slave Interface: eno4
MII Status: up
Speed: 1000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 6c:ae:8b:40:65:14
Slave queue ID: 0
```

### 链路绑定完成
创建不同vlan10 同理创建vlan12
``` bash
[root@node1 network-scripts]# vi ifcfg-br6.10
DEVICE="br6.10"
VLAN="yes"
ONBOOT="yes"
BOOTPROTO="none"
BRIDGE=brvlan6-10

[root@node1 network-scripts]# vi ifcfg-brvlan6-10
TYPE=bridge
BOOTPROTO=static
NAME=brvlan6-10
DEVICE=brvlan6-10
ONBOOT=yes
```

### 重启网络，并查看状态
``` bash
[root@node1 network-scripts]# systemctl restart network
[root@node1 network-scripts]# brctl show
 
[root@node1 network-scripts]# brctl show
bridge name	bridge id		STP enabled	interfaces
br6		8000.6cae8b406515	no		bond6
breno3		8000.6cae8b406513	no		eno3
							vnet0
brvlan6-10		8000.6cae8b406515	no		br6.10
brvlan6-11		8000.6cae8b406515	no		br6.11
							vnet1
brvlan6-12		8000.6cae8b406515	no		br6.12
virbr0		8000.525400de92ad	yes		virbr0-nic

```

### 虚机的挂在vlan
``` bash
brctl addif brvlan-10 vnet0
brctl addif brvlan-12 vnet1
brctl show
 
#vnet0 为vlan10下的ip 10.128.0.96
#vnet1 为vlan12下的ip 10.128.2.10
```

至此，环境搭建完毕,链路聚合，并且划分了不同的vlan