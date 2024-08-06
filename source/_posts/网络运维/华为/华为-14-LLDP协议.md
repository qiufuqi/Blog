---
title: 华为-14-LLDP协议
date: 2024-2-19
tags:
  - 华为
  - LLDP
categories: 
- 运维
- 华为
- LLDP
keywords: '华为,LLDP'
cover: https://qiufuqi.github.io/img/hexo/20230327084004.png
abbrlink: huawei_lldp
url: huawei_lldp
comments: false
---

[文章参考](https://support.huawei.com/enterprise/zh/doc/EDOC1100334405/13bbb5c3?idPath=24030814|21782164|21782167|259602657)
# LLDP简介
LLDP（Link Layer Discovery Protocol，链路层发现协议）是IEEE 802.1ab中定义的第二层发现（Layer 2 Discovery）协议。LLDP提供了一种标准的链路层发现方式，可以将本端设备的主要能力、管理地址、设备标识、接口标识等信息封装到LLDP报文中传递给邻居节点，邻居节点在收到这些信息后将其以标准MIB（Management Information Base，管理信息库）的形式保存起来，供NMS（Network Management System，网络管理系统）查询及判断链路的通信状况。
LLDP提供了一种标准的链路层发现方式。通过LLDP获取的设备二层信息能够快速获取相连设备的拓扑状态；显示出客户端、交换机、路由器、应用服务器以及网络服务器之间的路径；检测设备间的配置冲突、查询网络失败的原因。用户可以通过使用网管系统，对支持运行LLDP协议的设备进行链路状态监控，在网络发生故障的时候快速进行故障定位。

## LLDP基本原理
**LLDP可以将本地设备的信息发送给远端设备，本地设备将收到的远端设备信息以标准MIB的形式保存起来。**
LLDP协议规定设备的每个接口上都有四个MIB库，其中最主要的两个为LLDP Local System MIB（LLDP本地系统MIB）和LLDP Remote System MIB（LLDP远端系统MIB），分别存储着本端设备和邻居节点的状态信息，包括设备ID、接口ID、系统名称、系统描述、接口描述、设备能力、网络管理地址。
LLDP基本实现原理如下：
1. LLDP模块通过LLDP代理与设备上物理拓扑MIB、实体MIB、接口MIB以及其他MIB的交互，来更新自己的LLDP本地系统MIB，以及本地设备自定义的LLDP扩展MIB。
2. 将本地设备信息封装成LLDP帧发送给远端设备。
3. 接收远端设备发过来的LLDP帧，更新自己的LLDP远端系统MIB，以及远端设备自定义的LLDP扩展MIB。

通过LLDP代理收发LLDP帧，设备就很清楚地知道远端设备的信息，包括连接的是远端设备的哪个接口、远端设备的MAC地址等信息。
LLDP代理主要完成如下工作：
- 维护LLDP本地系统MIB信息。
- 向邻居节点发送LLDP帧，通告本端设备的状态信息。
- 识别并解析收到的邻居节点发送的LLDP帧，维护LLDP远端系统MIB的信息。
- LLDP本地系统MIB或LLDP远端系统MIB的信息发生变化时，向网管发送LLDP告警。

LLDPDU是封装在LLDP报文中的数据单元，封装了LLDP数据单元LLDPDU（LLDP Data Unit）的以太网报文称为LLDP报文。组成LLDPDU之前，设备先将本地信息封装成TLV（Type-Length-Value）格式，再由若干个TLV组合成一个LLDPDU，封装在LLDP报文的数据部分进行传送。
TLV是组成LLDPDU的最小单元，表示一个对象的类型、长度和信息。

## LLDP工作模式
LLDP有如下四种工作模式：LLDP功能启用后，默认的工作模式为Tx/Rx模式。
- Tx模式：只发送LLDP报文。
- Rx模式：只接收LLDP报文。
- Tx/Rx模式：既发送也接收LLDP报文。
- Disable模式：既不发送也不接收LLDP报文。

### LLDP报文的发送机制
一般情况下，启用LLDP功能后，**设备会周期性地向邻居节点发送LLDP报文**。如果设备的本地配置发生变化，则立即发送LLDP报文，将本地信息的变化情况尽快通知给邻居节点。为了防止本地信息的频繁变化而引起LLDP报文的大量发送，设备支持配置接口发送LLDP报文的延迟时间，**每发送一个LLDP报文后都延迟一段时间后再继续发送下一个报文**。

当设备发现一个新邻居（即接收到一个新的LLDP报文且本地没有保存该报文的发送方设备的信息），或者设备的LLDP功能由打开状态变为关闭，或者设备的接口状态由Down变为Up的时候，为了让其他设备尽快发现本设备，**设备支持快速发送机制，即将LLDP报文的发送周期缩短为1秒，并连续发送指定数量的LLDP报文后再恢复为正常的发送周期**。
### LLDP报文的接收机制
设备收到LLDP报文时，会对报文及其携带的TLV信息进行有效性检查，通过有效性检查后，将邻居信息保存到本地设备，并根据LLDPDU中携带的TTL（Time To Live，生存时间） TLV值，设置邻居信息在本地设备的老化时间，如果接收到的LLDPDU中的TTL值等于零，将立刻老化掉该邻居信息。

## LLDP缺省配置

# 启用LLDP
在启用LLDP之前，需完成以下任务：
- 设备与NMS之间路由可达，且SNMP相关参数已经配置完成。
- 设备上已经配置用于LLDP管理的IP地址。

## 启用LLDP功能
操作步骤:
1. 进入系统视图。
system-view
2. 启用全局的LLDP功能
lldp enable
3. （可选）使能LLDP管理地址的接口模式。
lldp management-address interface-mode
4. （可选）关闭接口的LLDP功能。
    - 进入接口视图。
    interface interface-type interface-number
    - 关闭接口的LLDP功能。
    lldp disable

缺省情况下，如果全局的LLDP功能未启用，接口的LLDP功能也未启用；如果全局的LLDP功能已启用，接口的LLDP功能也启用。
LLDP功能有两个开关，一个是全局开关，一个是接口下的开关，两者有如下的关系：
- 缺省情况下，启用全局的LLDP功能后，设备上所有接口的LLDP功能都处于打开状态。
- 关闭全局的LLDP功能后，设备上所有接口的LLDP功能都处于关闭状态。
- 只有全局的LLDP功能和接口下的LLDP功能都处于打开状态时，相应的接口才能够发送和接收LLDP报文。
- 全局的LLDP功能处于关闭状态时，打开和关闭接口的LLDP功能的命令都是无效的。

**接口的LLDP功能只能在物理接口上进行配置，逻辑接口不支持配置该功能。**
5. （可选）配置接口的LLDP工作模式。
    - 进入接口视图。
    interface interface-type interface-number
    - 配置接口的LLDP工作模式。
    lldp admin-status { tx | rx | txrx }
配置接口工作在指定的工作模式下，可以有效减少网络中LLDP报文的数量，降低系统负担，保证其他业务的正常运行。

## 配置管理地址
（可选）配置LLDP管理IP地址。管理IP地址是指被携带在LLDP报文中的Management Address TLV中，供网管系统标识设备，并进行网络管理的IP地址。管理IP地址可以明确地标识一台设备，从而有利于网络拓扑的绘制，便于网络管理。
配置LLDP管理IP地址有如下两种方式，只能选择一种方式配置：
- 通过lldp management-address命令为设备手工指定LLDP管理IP地址。
- 通过lldp management-address bind命令配置LLDP管理IP地址和指定接口的绑定关系，**设备优选绑定接口的IP地址为管理IP地址**。

LLDP管理IP地址的选取优先级：
- 用户通过lldp management-address命令配置了管理IP地址，但未配置管理IP地址和接口的绑定关系时，配置的管理IP地址的优先级最高，设备优选该IP地址作为管理IP地址。
- 用户未通过lldp management-address命令配置管理IP地址，但是配置了管理IP地址和接口的绑定关系时，设备将优选绑定接口的IP地址作为管理IP地址。
- 用户既未通过lldp management-address命令配置管理IP地址，又没有配置管理IP地址和接口的绑定关系时，系统将自动从IP地址列表中查找并指定一个IP地址作为LLDP的管理IP地址，如果没有找到缺省的IP地址，则用设备的桥MAC作为管理IP地址

操作步骤：
1. 进入系统视图。
system-view
2. 配置LLDP管理IP地址，请根据需要选择如下方式的一种。
   - 配置LLDP管理IPv4地址。
    lldp management-address ip-address
   - 配置LLDP管理IPv6地址。
    lldp management-address ipv6 IPv6address
   - 配置LLDP管理IPv4地址和接口的绑定关系。
    lldp management-address bind interface interface-type interface-number
   - 配置LLDP管理IPv6地址和接口的绑定关系。
    lldp management-address ipv6 bind interface interface-type interface-number

## 检查配置结果
已经完成LLDP基本功能的所有配置。

操作步骤：
- 执行命令display lldp local [ interface interface-type interface-number ]查看全局或指定接口的本地LLDP状态信息。
- 执行命令display lldp neighbor [ interface interface-type interface-number ]查看全局或指定接口的邻居节点的LLDP状态信息。
- 执行命令display lldp neighbor brief查看邻居节点的概要信息。
- 执行命令display lldp tlv-config [ interface interface-type interface-number ]查看全局或指定接口下配置的允许发送的可选TLV信息。
- 执行命令display lldp mdn local [ interface interface-type interface-number ]查看全部或者指定接口的MDN功能的配置和状态信息。
- 执行命令display lldp mdn neighbor [ interface interface-type interface-number ]查看全部或者指定接口的MDN邻居信息。

## LLDP告警功能
通过配置LLDP告警功能，设备会在邻居节点信息发生变化的时候，向NMS发送告警。

为了防止因为LLDP告警频繁发送而导致设备上的网络拓扑发生震荡，需要配置设备发送告警的延迟时间。设备发送LLDP告警的延迟时间的取值要适当，用户可以根据网络负载及时调节该参数。
- 取值较大时，能够有效避免设备上的网络拓扑发生震荡。但是，如果取值过大会导致NMS不能及时感知到邻居节点的状态变化，影响设备上的网络拓扑的及时更新。
- 取值较小时，能够及时更新设备上的网络拓扑。但是，如果取值过小会导致NMS频繁刷新邻居节点的状态信息，不仅会导致设备上的网络拓扑发生震荡，还增加了系统的负担，造成资源的浪费。

操作步骤：
1. 进入系统视图。
system-view
2. 启用LLDP告警功能。
snmp-agent trap enable feature-name lldp [ trap-name { hwlldpinterfaceremtableschange  | lldpremtableschange } ]
启用设备的LLDP告警功能后，当出现以下情况时，设备会向NMS发送告警。
- 接口邻居节点的状态信息发生变化，对应的告警为LLDP 1.3.6.1.4.1.2011.5.25.134.2.8 hwLldpInterfaceRemTablesChange。
- LLDP邻居节点的状态信息发生变化，对应的告警为LLDP 1.0.8802.1.1.2.0.0.1 lldpRemTablesChange。

**LLDP告警功能对全局起作用，控制设备上所有接口发送LLDP告警的能力。**
3. （可选）配置设备发送LLDP邻居信息变化告警的延迟时间。
lldp trap-interval interval
**缺省情况下，设备发送LLDP邻居信息变化告警的延迟时间是5秒。**