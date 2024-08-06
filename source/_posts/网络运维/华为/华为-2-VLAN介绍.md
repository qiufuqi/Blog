---
title: 华为-2-VLAN介绍
date: 2024-2-10
tags:
  - 华为
  - VLAN
  - 端口
categories: 
- 运维
- 华为
- VLAN
keywords: '华为,VLAN,端口'
cover: https://qiufuqi.github.io/img/hexo/20230327084004.png
abbrlink: huawei_vlan
url: huawei_vlan
comments: false
---

**VLAN 虚拟局域网**
虚拟局域网vlan主要用于隔离广播域，不受地域限制，同一个vlan的设备才能进行二层通信。[文章参考](https://blog.csdn.net/weixin_55807049/article/details/122673738)

# VLAN标签
vlan tag 在报文中添加标识vlan的字段 IEEE 802.1Q协议规定，在以太网的数据帧中添加4字节的vlan标签，简称tag
缩小广播域，隔离广播域 
- VLANID由12bit组成，所以就有2^12=4096个VLANID
- VLANID的取值范围为0-4095，一共4096个；其中0和4095保留不用，可使用的VLANID范围为：1-4094
（温馨提示：VLANID 1尽量别用，因为不同厂商对VLANid 1的默认配置不同，在使用过程中，很可能导致其他协议无法正常使用！！慎重！！）

# VLAN划分
基于接口划分，基于mac地址划分，基于IP子网划分，基于协议划分，基于策略划分

## 基于接口划分
网路管理员给每个接口划分不同的PVID，将该接口划入对应的VLAN；如果一个数据帧进入是没有带vlan标签（无标记帧），该数据帧就会被打上接口指定PVID对应的tag；**取值范围1-4094**

## 接口类型
华为交换机端口类型有三种：
- Access：交换机上常用来连接用户PC，服务器等终端设备的接口。Access接口所连接的这些设备的网卡往往只能收发无标记帧,Access口只能加入一个VLAN；
- Trunk：Trunk接口允许多个VLAN的数据帧通过，这些数据帧通过802.1Q Tag实现区分。Trunk通常用于交换机之间的互联，也用户连接路由器防火墙等设备的子接口；
- Hybrid：Hybrid接口与Trunk接口类似，也允许多个VLAn的数据帧通过，这些数据帧通过802.1Q Tag实现区分，用户可以灵活指定Hybrid接口在发送某个或某些VLAn数据帧时是否携带Tag

### access端口
![](https://qiufuqi.github.io/img/hexo/20240210113546.png)
  - 收不带标签：当access接收不带标签的数据帧时，则打上自己接口的PVID，接口PVID是多少，该数据帧就属于哪个VLAN
  - 收带标签：当access接收带标签的数据帧时，则查看该数据帧的VLAN ID和接口的PVID是否一致，一致则接收该数据帧，不一致则丢弃
  - 发不带标签：access只能发送不带标签的数据帧，当**数据帧中带有的VLAN ID和接口的PVID一致时，剥离标签发送**，不一致则丢弃
```bash
[SW1]vlan batch 10
Info: This operation may take a few seconds. Please wait for a moment...done.
[SW1]int g0/0/1
[SW1-GigabitEthernet0/0/1]port link-type access
[SW1-GigabitEthernet0/0/1]port default vlan 10
```

### trunk接口
允许通行列表，决定是否能放行。PVID不影响通行，只影响是否剥离标签或直接带着原有的标签发送。干道链路，用于交换机之间互联，在一条链路上可以传输多个VLAN的数据帧，trunk接口在于允许同性列表，只要允许通行必定能被接收/发送
![](https://qiufuqi.github.io/img/hexo/20240210114354.png)
  - 收不带标签：当trunk接收不带标签的数据帧时，首先打上该接口的PVID，然后看是否允许通行，如果允许则从该接口接入，否则丢弃
  - 收带标签：当trunk接收带标签的数据帧时，只看该VLAN ID是否允许通行，如果允许则接收
  - 发数据帧：当trunk接口发送数据帧时，先看是否允许通行，如果允许则发送，不允许则丢弃
    在允许的前提下，如果数据帧的VLAN ID和接口的PVID一致，则**剥离标签发出**，如果不一致则直接发
```bash
[SW1]int g0/0/2
[SW1-GigabitEthernet0/0/2]port link-type trunk 
[SW1-GigabitEthernet0/0/2]port trunk pvid vlan 10	      # 将接口PVID标识更改为vlan 10 （发送vlan10的数据时，会剥离标签，成不带标签帧）
[SW1-GigabitEthernet0/0/2]port trunk allow-pass vlan 10 20 
```

### hybrid接口
![](https://qiufuqi.github.io/img/hexo/20240210115753.png)
hybrid接受帧的处理方式与trunk一致；发送帧的处理方式，管理员来自己设置。hybrid接口是华为默认接口，没配置之前，**所有端口默认都是hybrid接口**。
hybrid接口的untagged和tagged：接收帧时不影响； 只影响发送帧 即只有发送数据帧时才会考虑是否tagged或untagged
```bash
[SW1]int g0/0/2
[SW1-GigabitEthernet0/0/2]port link-type hybrid 
# 将接口PVID标识更改为vlan 10
[SW1-GigabitEthernet0/0/2]port hybrid pvid vlan 10	    

# 允许放行vlan10 20 并且发送数据时保留标签发送；允许放行vlan 30 40 并且发送数据时剥离标签发送
[SW1-GigabitEthernet0/0/2]port hybrid tagged vlan 10 20
[SW1-GigabitEthernet0/0/2]port hybrid untagged vlan 30 40
```
### 端口收发总结
![](https://qiufuqi.github.io/img/hexo/20240210120254.png)

