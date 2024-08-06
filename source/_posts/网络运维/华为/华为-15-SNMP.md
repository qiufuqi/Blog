---
title: 华为-15-SNMP协议
date: 2024-2-20
tags:
  - 华为
  - SNMP
categories: 
- 运维
- 华为
- SNMP
keywords: '华为,SNMP'
cover: https://qiufuqi.github.io/img/hexo/20230327084004.png
abbrlink: huawei_snmp
url: huawei_snmp
comments: false
---

[文章参考华为官网](https://support.huawei.com/enterprise/zh/doc/EDOC1100279010/60bd56a0#ZH-CN_CONCEPT_0000001124849544)

# SNMP简介
简单网络管理协议SNMP（Simple Network Management Protocol）是广泛用于UDP网络的网络管理标准协议。SNMP提供了一种通过运行网络管理软件的中心计算机（即网络管理工作站NMS）来管理网元的方法。共有三个版本SNMPv1、SNMPv2c和SNMPv3，用户可以根据情况选择同时配置一个或多个版本。

网管通过在被管理设备中运行的Agent客户端上执行get和set操作来管理设备上的节点，设备上的节点由MIB（Management Information Base）来唯一标识。

在大型网络中，设备发生故障时，由于设备无法主动上报故障，导致网络管理员无法及时感知、及时定位和排除故障，从而导致网络的维护效率降低，维护工作量大大增加。为了解决这个问题，设备制造商已经在一些设备中提供了网络管理的功能，这样网管就可以远程查询设备的状态，同样设备能够在特定类型的事件发生时向网络管理工作站发出警告。

SNMP就是规定网管站和设备之间如何传递管理信息的应用层协议。SNMP定义了网管管理设备的几种操作，以及设备故障时能向网管主动发送告警。

# SNMP三类角色
在网管通过SNMP协议管理设备过程中，定义了三类角色：
- 网管站：向被管理设备发送各种查询报文，以及接收被管理设备发送的告警。
- Agent：被管理设备上的网管进程。Agent有如下功能：
  - 接收、解析来自网管站的查询报文。
  - 根据报文类型对管理变量进行Read或Write操作，并生成响应报文，返回给网管站。
  - 根据各协议模块对告警触发条件的定义，在达到触发条件后，如进入、退出系统视图或设备重新启动等，相应的模块通过Agent主动向网管站发送告警，报告所发生的事件。
- 被管理设备：接受网管的管理，产生和主动上报告警。
![](https://qiufuqi.github.io/img/hexo/20240220205311.png)

# SNMP操作
| 操作 | 功能 |
| :------------- | :------------- |
| GetRequest | 从某变量中取值，获取设备某功能节点状态，是网管向设备发出的请求 | 
| GetNextRequest | 在MIB表项中取下一项值，获取设备某功能的另一节点状态，是网管向设备发出的请求|
| GetResponse | 对GetRequest、GetNextRequest、SetRequest的响应操作，是被管理设备向网管的回应，是由代理进程发出的|
| GetBulk | 该操作相当于连续执行多次GetNext操作，是网管向设备发出的请求|
| SetRequest | 设置具体变量的值，对设备功能节点状态进行调整，是网管向设备发出的指令|
| Trap | **报告事件信息，是设备主动向网管报告事件**|
| Inform | **报告事件信息，需要网管进行接收确认，是设备主动向网管报告事件**|

## Trap操作
**Trap是被管理设备主动向NMS发送的不经请求的信息**，用于报告一些紧急的重要事件（如被管理设备重新启动等）
设备端的模块由于达到模块定义的告警触发条件，通过Agent向网管工作站发送Trap消息，告知设备侧的出现的情况，这样便于网络管理人员及时对网络中出现的情况进行处理。
**网管使用162端口接收Agent发送的Trap报文，报文承载协议为UDP，不需要网管进行接收确认**。





# 配置SNMPv3的基本功能
操作步骤
1. 进入系统视图。
system-view
2. （可选）配置SNMP密码的最小长度。
snmp-agent password min-length min-length
配置了该命令后，用户配置SNMP密码时，密码长度必须大于等于配置的SNMP密码的最小长度。

3. （可选）启动SNMP Agent服务。
snmp-agent
缺省情况下，没有启动SNMP Agent服务。执行任意snmp-agent的配置命令（无论是否含参数）都可以触发SNMP Agent服务启动，故该步骤可选。

4. （可选）修改SNMP Agent侦听的端口号。
snmp-agent udp-port port-number
缺省情况下，SNMP Agent侦听的端口号为161。如果不执行此步骤，会使用缺省端口号进行侦听。

5. 配置SNMP的协议版本。
snmp-agent sys-info version v3
缺省情况下，使能SNMPv3。

6. 配置SNMP用户组。
snmp-agent group v3 group-name { authentication | privacy | noauthentication } [ read-view read-view | write-view write-view | notify-view notify-view ] * [ acl { acl-number | acl-name } ]
当网管和设备处在不安全的网络环境中时，比如容易遭受攻击，建议用户配置参数authentication或privacy，使能数据的认证和加密功能。
用户可以选择的认证加密模式如下：
- 不配置authentication和privacy参数或配置noauthentication参数：不认证不加密。适用于网络环境安全，且管理员比较固定的情况下。
- 配置authentication参数：只认证不加密。适用于网络环境安全，但管理员个数多，管理员对设备交叉操作比较频繁的情况下。通过认证可以限制拥有权限的管理员才可以访问该设备。
- 配置authentication和privacy参数：既认证又加密。适用于网络环境不太安全，管理员交叉操作多的情况下。通过认证和加密既可以限制特定的管理员访问设备，并且使网络数据以加密形式发送，避免网络数据被窃取，造成关键数据泄露。

希望网管在指定视图下具有只读权限时（比如级别比较低的管理员），使用read-view参数。
希望网管在指定视图下具有读写权限时（比如级别比较高的管理员），使用write-view参数。

当希望过滤不相关告警并配置被管理设备只向网管发送指定MIB节点的告警信息，使用notify-view参数。如果配置了该参数，只有notify-view视图下的MIB节点的告警会发送到网管。

7. （可选）设置本地SNMP实体的引擎ID。
snmp-agent local-engineid engineid
缺省情况下，系统采用内部算法自动生成一个设备引擎ID，包含公司的“企业号＋设备信息”。其中，系统采用设备的管理网口MAC地址作为引擎ID的“设备信息”。