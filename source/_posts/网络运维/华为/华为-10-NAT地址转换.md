---
title: 华为-10-NAT地址转换
date: 2024-2-13
tags:
  - 华为
  - NAT
categories: 
- 运维
- 华为
- NAT
keywords: '华为,NAT,地址转换'
cover: https://qiufuqi.github.io/img/hexo/20230327084004.png
abbrlink: huawei_nat
url: huawei_nat
comments: false
---
NAT就是通过将一个IP地址转换为另一个IP地址。NAT不仅可以节省IP地址空间，而且能够提高网络的安全性（外部网络的主机无法直接与内部主机进行通信）。为了让使用私网IP地址的主机能够访问互联网，需要将私有IP地址转换为公有IP地址，这类设备通常为网络的出口设备，如路由器或防火墙。
- 公有IP地址：由专门的机构进行管理和分配，可以直接再互联网上进行通信，需要付费使用
- 私有IP地址：任何人可以随意使用，无法直接在互联网进行通信，无须付费

| 类别 | IP地址段 | IP地址数量 | 描述 |
| :------------- | :------------- | :------------- | :------------- |
| A类 | 10.0.0.0~10.255.255.255 | 16777216 | 单个A类网络 |
| B类 | 172.16.0.0~172.31.255.255 | 1048576 | 16个连续的B类网络 |
| C类 | 192.168.0.0~192.168.255.255 | 65536 | 256个连续的C类网络 |
# NAT类型
NAT共有4中类型：
- 静态NAT：将一个私有IP地址映射为一个固定公有IP地址；需要管理员通过使用命令来绑定私有IP地址与公有IP地址
- 动态NAT：将一个私有IP地址映射为一个非固定公有IP地址；将**私有IP地址与一个地址池中的公有IP地址进行动态关联，**管理员需要在地址池中指定可以使用的公有IP地址范围即可，无须静态配置一对一的映射。
  公有地址池中的IP数量应不少于需要访问互联网的内部主机数量，否则当公有地址池中的IP地址耗尽后，其他内部主机将无法访问互联网
- NAPT和Easy IP：Network Address Port Translation 将多个私有IP地址映射为一个或者多个公有IP地址，二者工作原理相同。在Easy IP的配置中，NAT设备使用本地连接互联网的接口IP地址作为公有IP地址，为内部主机提供互联网连接；NAPT配置中，可以像配置动态NAT一样设置地址池，地址池中的公有IP数量可以远远小于内部需要访问互联网的主机数量
- NAT Server：与静态NAT相似，将一个私有IP地址映射为一个固定公有IP地址，NAT Server映射还添加了端口号信息，使内部主机能够对外提供服务

## 静态NAT
静态NAT的特点是每个私有IP地址都有一个与之绑定的公有IP地址。在静态NAT中，NAT设备保存着一个NAT映射表，记录着私有IP和公有IP的映射关系，并且映射表会与NAT设备的某个端口相关联。
- 正向查找：NAT设备转发数据包时，设备根据数据包的源IP地址查找NAT映射表中的私有IP地址，并将数据包中的源IP地址转换为对应条目中映射的公有IP地址 （内网发向外网）
- 反向查找：NAT设备接收数据包时，设备根据数据包中的目的IP地址查找NAT映射表中的公有IP地址，并将数据包中的目的IP地址转换为对应条目中映射的私网IP地址（外网发向内网）

配置方法有两种，任选一种即可：
- 接口视图下配置映射关系
  **nat static global {global-address} inside {host-address}** ：global配置公网地址，inside配置私网地址
- 系统视图下配置映射关系，并在接口视图下启用静态NAT
  **系统视图下配置：nat static global {global-address} inside {host-address}**
  **接口视图下启用：nat static enable**

## 动态NAT
在动态NAT中，NAT设备不仅能对地址池中的公有IP地址的使用状态进行标记，还需要知道公有IP地址的分配信息。
- 正向查找：NAT设备转发数据包时，设备根据数据包的源IP地址执行NAT转换，从NAT地址池中选择一个未使用的公有IP地址，在NAT映射表中添加一个条目，并将数据包的源IP地址转换为公有IP地址
- 反向查找：NAT设备接收数据包时，设备根据数据包的目的IP地址对NAT映射表进行反向查找，并根据查找结果将数据包的目的IP地址转换为私有的IP地址

配置步骤：
- 指定公有IP地址：使用NAT地址池指定公有IP地址
  **系统视图下配置：nat address-group group-index start-address end-address**
  地址池编号，IP地址范围的起始地址和结束地址
- 指定私有IP地址：使用ACL指定需要被转换的私有IP地址。因为只需要指定源IP地址，所以使用基本ACL即可。当ACL用于NAT时，需要**配置permit行为，并且匹配源IP地址**。所有匹配ACL的数据包需要被转换，未匹配ACL规则的数据包按照原始方式进行处理
  **系统视图下配置：acl number** 基本ACL编号：2000-2999，高级ACL编号：3000-3999
  **ACL视图下配置：rule permit source-address source-wildcard** 配置被转换的私有IP地址范围
  **接口视图下配置：nat outbound acl-number address-group group-index [no-pat]** 命令中acl-number指明了私有IP地址，address-group指明了公有IP地址。**no pat 指不进行端口转换，只针对IP转换**

## NAPT和Easy IP
在使用NAPT为内部主机提供互联网连接时，可以根据公有IP地址的情况来选择部署方式
- NAPT：当配置了NAT设备连接ISP（Internet Service Provider 互联网服务提供商）的接口IP地址和其他应用后，还有空闲的IP地址，可以选择根据地址池来指定公有IP地址的方式来为内部主机提供可用的公用IP地址
- Easy IP：当配置了NAT设备连接ISP（Internet Service Provider 互联网服务提供商）的接口IP地址和其他应用后，已经没有公有IP地址，可以使用Easy IP的方式，复用NAT设备连接ISP的出口IP地址为内部主机提供互联网连接

NAPT从地址池中选择公有IP地址进行转换时，同时对端口号进行转换，实现一对多转换，有效提升IP地址的利用率。
**配置方法和动态NAT相同，不包含no-pat，即启用端口转换**
Easy IP的工作原理和NAPT相同，结合IP地址和端口号进行转换，Easy IP无须配置地址池，只需要在ACL中指定需要被转换的私网IP地址，并在接口上进行关联即可
**系统视图下配置：acl number** 基本ACL编号：2000-2999，高级ACL编号：3000-3999
  **ACL视图下配置：rule permit source-address source-wildcard** 配置被转换的私有IP地址范围
  **接口视图下配置：nat outbound acl-number**

## NAT Server
NAT Server既可以让内部服务器能够向外提供服务，也可以限制外部主机可访问端口来保障安全性。NAT Server在静态NAT的基础上添加端口号，将**私有IP地址+端口号和公有IP地址+端口号**映射在一起。
配置方法：
**接口视图下启用：nat server protocol {tcp|udp} global global-address global-port inside host-address host-port** 通过tcp或udp来确定传输协议，在global部分设置公网IP+端口，在inside部分设置私网IP+端口

# NAT拓扑实验
模拟情况：AR1是企业网关路由器，G0/0/1连接内网服务器，g0/0/0连接外网，AR2为外网，环路IP地址（8.8.8.8/32）假定为互联网
![](https://qiufuqi.github.io/img/hexo/20240214204310.png)
基础配置：无须在AR2中配置任何路由信息，一般情况下ISP无须了解客户内网IP规划。
``` bash
[R1]int g0/0/0
[R1-GigabitEthernet0/0/0]ip add 102.10.200.5 29
[R1-GigabitEthernet0/0/0]quit
[R1]int g0/0/1
[R1-GigabitEthernet0/0/1]ip add 192.168.0.1 24
[R1-GigabitEthernet0/0/1]quit
[R1]ip route-static 0.0.0.0 0.0.0.0 102.10.200.6

[R2]int g0/0/0
[R2-GigabitEthernet0/0/0]ip add 102.10.200.6 29
[R2-GigabitEthernet0/0/0]quit
[R2]int LoopBack 0
[R2-LoopBack0]ip add 8.8.8.8 32
```
## 静态NAT配置
配置静态NAT时，需要保证global-address和host-address是唯一没有重复的，要避免使用设备接口地址。102.10.200.5和102.10.200.6用于AR1和AR2的直连链路，因此内网可以使用102.10.200.1-102.10.200.4之间4个IP地址访问Internet（8.8.8.8）
静态NAT配置有两个缺点：1.无法实现公有IP的复用，一个公有IP地址只能支持一台内网PC访问Internet；2.将内部主机暴露在公网中
**nat static global global-address inside host-address**
**display nat static** 查看静态NAT配置信息
``` bash
# 在AR1的g0/0/0接口上为PC1配置静态NAT，此时只有PC1能够访问外网（8.8.8.8）
[R1]int g0/0/0
[R1-GigabitEthernet0/0/0]nat static global 102.10.200.1 inside 192.168.0.10

# 或者系统视图下配置转换，接口视图下启用
[R1]nat static global 102.10.200.1 inside 192.168.0.10
[R1]int g0/0/0
[R1-GigabitEthernet0/0/0]nat static enable

# 同理将PC的内网地址也做映射，即可实验两台PC都能够访问互联网
nat static global 102.10.200.2 inside 192.168.0.20

[R1]display nat static
  Static Nat Information:
  Global Nat Static 
    Global IP/Port     : 102.10.200.2/---- 
    Inside IP/Port     : 192.168.0.20/----
    Protocol : ----     
    VPN instance-name  : ----                            
    Acl number         : ----
    Netmask  : 255.255.255.255 
    Description : ----

  Global Nat Static 
    Global IP/Port     : 102.10.200.1/---- 
    Inside IP/Port     : 192.168.0.10/----
    Protocol : ----     
    VPN instance-name  : ----                            
    Acl number         : ----
    Netmask  : 255.255.255.255 
    Description : ----

  Total :    2
```
## 动态NAPT配置
使用NAPT（Network Access Port Translation 网络地址端口转换）来实现多对一的地址转换。NAPT使用“IP地址+端口号”的形式实现转换，使多个内网主机能够共用一个公有IP访问Internet。
``` bash
# 先清理静态NAT转换
[R1]undo nat static global 102.10.200.2 inside 192.168.0.20 netmask 255.255.255.255
[R1]undo nat static global 102.10.200.1 inside 192.168.0.10 netmask 255.255.255.255
[R1]int g0/0/0
[R1-GigabitEthernet0/0/0]undo nat static enable
```
配置步骤按照以下顺序进行：
- 配置ACL匹配私有IP地址：根据需要使用基本ACL（2000-2999）或高级ACL（3000-3999），当ACL的规则为permit时，表示设备需要对匹配的源IP地址执行转换操作，未匹配的不执行任何操作
- 配置动态NAT使用地址池：转换后的公有IP地址范围
  **nat address-group group-index start-address end-address**
- 进入接口NAT视图配置NAT：将ACL和动态IP地址池进行关联，其中ACL定义了需要转换的内网IP地址，动态IP地址池定义了转换后的公有IP地址
  **nat outbound acl-number address-group group-index [no pat]**
  **动态NAT转换：no pat指不进行端口转换，只针对IP转换**

**display nat address-group** 查看设备NAT地址池信息
**display nat outbound** 查看NAT转换表
**display nat session all** 查看NAT转发表
```bash
[R1]acl 2000
[R1-acl-basic-2000]rule permit source 192.168.0.0 0.0.0.255
[R1-acl-basic-2000]quit
[R1]nat address-group 1 102.10.200.1 102.10.200.4
[R1]int g0/0/0
[R1-GigabitEthernet0/0/0]nat outbound 2000 address-group 1

[R1]display nat address-group 
 NAT Address-Group Information:
 --------------------------------------
 Index   Start-address      End-address
 --------------------------------------
 1        102.10.200.1     102.10.200.4
 --------------------------------------
  Total : 1
[R1]display nat outbound 
 NAT Outbound Information:
 --------------------------------------------------------------------------
 Interface                     Acl     Address-group/IP/Interface      Type
 --------------------------------------------------------------------------
 GigabitEthernet0/0/0         2000                              1       pat
 --------------------------------------------------------------------------
  Total : 1

[R1]display nat session all
  NAT Session Table Information:

     Protocol          : ICMP(1)
     SrcAddr   Vpn     : 192.168.0.20                                   
     DestAddr  Vpn     : 8.8.8.8                                        
     Type Code IcmpId  : 0   8   45822
     NAT-Info
       New SrcAddr     : 102.10.200.4   
       New DestAddr    : ----
       New IcmpId      : ----

     Protocol          : ICMP(1)
     SrcAddr   Vpn     : 192.168.0.10                                   
     DestAddr  Vpn     : 8.8.8.8                                        
     Type Code IcmpId  : 0   8   45822
     NAT-Info
       New SrcAddr     : 102.10.200.3   
       New DestAddr    : ----
       New IcmpId      : ----

```

## Easy IP配置
使用一个公有的IP地址来连接ISP，以支持内网主机方位Internet。配置和动态NAT比较类似，**无须配置公有IP地址池**，使用出接口的IP地址作为转换后的IP地址
``` bash
# 删除多余配置
[R1]undo acl 2000
[R1]undo nat address-group 1
[R1]int g0/0/0
[R1-GigabitEthernet0/0/0]undo nat outbound 2000 address-group 1
```
配置步骤按照以下顺序进行：
- 配置ACL匹配私有IP地址：根据需要使用基本ACL（2000-2999）或高级ACL（3000-3999），当ACL的规则为permit时，表示设备需要对匹配的源IP地址执行转换操作，未匹配的不执行任何操作
- 进入接口NAT视图配置NAT：通过ACL来指定哪些私有IP地址需要转换
  **nat outbound acl-number**

``` bash
[R1]acl 2000
[R1-acl-basic-2000]rule permit source 192.168.0.0 0.0.0.255
[R1]int g0/0/0
[R1-GigabitEthernet0/0/0]nat outbound 2000

# 当前g0/0/0的公有IP地址为：102.10.200.5，类型展示为easyip
[R1]display nat outbound 
 NAT Outbound Information:
 --------------------------------------------------------------------------
 Interface                     Acl     Address-group/IP/Interface      Type
 --------------------------------------------------------------------------
 GigabitEthernet0/0/0         2000                   102.10.200.5    easyip  
 --------------------------------------------------------------------------
  Total : 1
```
## NAT Server配置
假定内网服务器需要提供对外http服务，配置和静态NAT相似，增加了端口号。
在接口视图模式下配置命令如下：
**nat server protocol {tcp|udp} global glo-address glo-port inside host-address host-port**
在IP地址映射的情况下增加了端口映射。
**display nat server**：查看生成的配置
``` bash
# 参考实例  接口视图下  www代表80端口
nat server protocol tcp global 102.10.200.4 www inside 192.168.0.10 www
```