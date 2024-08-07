---
title: Cisco路由器access-list访问控制
date: 2023-05-18
tags:
  - 交换机
  - Cisco
categories: 
- 运维
- 交换机
- Cisco
keywords: '交换机,Cisco,访问控制'
cover: https://qiufuqi.github.io/img/hexo/20230406153459.png
abbrlink: cisco_access
url: cisco_access
comments: false
---

什么是ACL？
访问控制列表（Access Control List，ACL） 是路由器和交换机接口的指令列表，用来控制端口进出的数据包。ACL适用于所有的被路由协议，如IP、IPX、AppleTalk等。这张表中包含了匹配关系、条件和查询语句，表只是一个框架结构，其目的是为了对某种访问进行控制。
list number的范围在0～99之间，这表明该access-list语句是一个普通的标准型IP访问列表语句。

**华为默认最后未被匹配的路由都permit ，思科 默认最后都未被匹配的路由是deny**

标准型IP访问列表的格式
---- 标准型IP访问列表的格式如下：
---- access-list[list number][permit|deny][source address][address][wildcard mask][log]
---- 下面解释一下标准型IP访问列表的关键字和参数。首先，在access和list这2个关键字之间必须有一个连字符"-"；

允许指定特定的主机，可以增加一个通配符掩码0.0.0.0或者使用host（host代表通配符），any则代表所有的
``` bash
# 指定主机
access-list 1 permit 192.168.3.10 0.0.0.0 
access-list 1 permit host 192.168.3.10 　
access-list 1 permit any any


access-list 1 permit 192.168.1.33 0.0.0.0     #其中1表示ACL的序号, 匹配和允许来 自192.168.1.33的流量
# 指定网络地址
access-list 1 permit 192.168.1.0 0.0.0.255    #匹配和允许来自192.168.1.0/24网段的流量, 注意这里使用的是反掩码, 0 表示比配, 1 表示任意

access-list 1 deny 172.10.0.0 0.0.255.255     #禁止172.10.0.0/16 网段的流量
access-list 1 permit 0.0.0.0 255.255.255.255  #允许所有的流量注意:

** 每一个access-list之后，思科都会在你配置后加上一句 access-list 1 deny any any 即拒绝所有**
例1:#access-list 1 permit 192.168.1.33 0.0.0.0 如果你只配置了上面这条命令, 那么在思科的路由器中, 表示抓取 192.168.1.33这条路由, 并拒绝这条路由之外的全部路由
例2:如果你想要拒绝一条路由, 放过所有的其他路由, 需要按照下面的方式配置
#access-list 1 deny 192.168.1.33 0.0.0.0 
#access-list 1 permit any any
在来看看扩展的ACL#access-list 100 deny ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255 // 其中192.168.1.0 代表源IP, 192.168.2.0代表目的IP

```
试验拓扑，具体接口配置如拓扑所示
![](https://qiufuqi.github.io/img/hexo/20230519160426.png)

实验结果1：
在router 0上配置ACL 应用到Fa0/0接口
access-list 20 permit host 172.16.30.3 

interface FastEthernet0/0
ip address 172.16.30.1 255.255.255.0
duplex auto
speed auto
ip access-group 20 in 
则 只有172.16.30.3 的主机能够访问 192.168.10.2 的主机，172.16.30.2 的主机不能访问。

实验结果2：
在router 0上配置ACL 应用到Fa1/0接口
access-list 20 permit host 172.16.30.3 

interface FastEthernet1/0
ip address 192.168.10.1 255.255.255.0
ip access-group 20 in （在该接口本应该为out ,配置为in 后 对应的控制方向就是控制源地址192.168.10.2的路由，而access-list中配置源为172.16.30.3 显然不匹配
故PC1(IP为192.168.10.1 )出去的流量都被拒绝）
duplex auto
speed auto
则 PC1 (IP为192.168.10.1 )ping PC0 与Laptop0都不通。


实验结果3：
在router 0上配置ACL 应用到Fa0/1接口
access-list 20 permit host 172.16.30.3
access-list 20 permit host 192.168.10.2

interface FastEthernet1/0
ip address 192.168.10.1 255.255.255.0
ip access-group 20 in 

放开源 192.168.10.2 后， PC1(IP为192.168.10.1 )ping PC0 与Laptop0都通。





示例参考：
配置的访问列表是允许来自网络172.16.2.0的流量
``` bash
Router# 
Router#configure terminal 
Router(config)# 
Router(config)#access-list 1 permit 172.16.2.0 0.0.0.255
```
禁止来自网络172.16.3.0的流量!!!
``` bash
Router#configure terminal 
Router(config)# 
Router(config)#access-list 2 deny 172.16.3.0 0.0.0.255 
Router(config)#access-list 2 permit any 
Router(config)#int s0 
Router(config-if)#ip access-group 2 out 
```



