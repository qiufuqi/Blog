---
title: vCenter虚拟机迁移问题
date: 2023-04-10
tags:
  - EXSI
  - vCenter
categories: 
- 运维
- EXSI
- vCenter
- 迁移
keywords: 'Exsi,vCenter,迁移'
cover: https://qiufuqi.github.io/img/hexo/20220922143047.png
abbrlink: exsi_vCenter_remove
comments: false
---

**VCSA迁移虚拟机**
可使用热迁移或冷迁移将虚拟机从一个主机或存储位置移至另一位置，可使用 vMotion 将已打开电源的虚拟机从主机上移开，以便执行维护、平衡负载、并置相互通信的虚拟机、将多个虚拟机分离以最大限度地减少故障域、迁移到新服务器硬件等等。

**冷迁移** 您可将已关闭电源或已挂起的虚拟机移至新主机。您可选择将已关闭电源或已挂起虚拟机的配置文件和磁盘文件重定位到新的存储位置。您也可使用冷迁移将虚拟机从一个数据中心移至另一个数据中心。要执行冷迁移，您可手动移动虚拟机或设置调度的任务。

**热迁移** 根据您使用的迁移类型是 vMotion 还是 Storage vMotion，您可以将打开的虚拟机迁移到其他主机，或者将其磁盘或文件夹迁移到其他数据存储，而不破坏虚拟机的可用性。vMotion 也称为“实时迁移”或“热迁移”。

一般来讲，冷迁移速度更快。

您不能在不同的数据中心之间移动已打开电源的虚拟机。
注： 复制虚拟机是指创建新的虚拟机，并不是迁移形式。通过克隆虚拟机或复制其磁盘和配置文件可以创建新的虚拟机，克隆并不是迁移的一种形式。

# 冷迁移
冷迁移即虚拟机关机进行迁移，一般冷迁移的速度更快。
# 热迁移
热迁移即虚拟机不关机，保持服务的情况下进行迁移，但是有时候会遇到CPU内核版本不兼容的情况，需要特殊处理。


## 不同CPU迁移问题
参考：https://blog.51cto.com/xiaoyuanzheng/5611330
在esxi 6.7以前的版本，有一句话，叫做vm要在线迁移要保证cpu型号是一致的，不然没办法进行迁移。这个对于多次购买不同型号服务器的机器来说是很痛苦的。
所以vmware从6.7版本开始支持跨cpu进行热迁移了，只需要开启一个叫evc的功能。（6.7以前没这个功能）
PS:如果你的vm版本是低于6.7的怎么办？答：把vm迁移到esxi 6.7或后面的版本，然后右键vm——兼容性——升级虚拟机兼容性，升级上去就行了。
创建一个集群，开启EVC 选择 Intel® "Westmere" Generation（具体看各自情况，兼容不同CPU，选择也不同，多试试几个）
![](https://qiufuqi.github.io/img/hexo/20230410142135.png)

服务器加入VMware集群或启用EVC时报错CPU缺少必要的功能
故障现象：
服务器加入集群或启用EVC模式时，报错：
主机的CPU硬件应支持集群当前的增强型vMotion兼容性模式，但主机现在缺少某些必要的CPU功能。请检查主机的BIOS配置，确保未禁用必要的功能（例如Intel的XD、VT、AES或PCLMULQDQ，或者AMD的NX）
![](https://qiufuqi.github.io/img/hexo/20230410151027.png)
vMotion功能还依赖于Mwait，然而2017年4月8号之后生产的通用双路M4机器已经默认关闭了此功能，请开启。
Socket Configuration => IIO Configuration => Monitor/Mwait Support
另外，打开Mwait后相当于服务器启用了休眠功能，请登录到Vmware系统下，将电源选项设置为高性能模式。
![](https://qiufuqi.github.io/img/hexo/20231205115139.png)
[参考](http://www.4008600011.com/archives/419#10_VMwareEVCCPU)

## 安全策略问题
错误信息如下：”当前已连接的网络接口”“Network adapter 1”无法使用网络“xxxx”，因为 “在目标主机上为目标网络配置的卸载或安全策略不同于在源主机上为源网络配置的卸载或安全策略”。
![](https://qiufuqi.github.io/img/hexo/20230410100719.png)
解决办法：
1、确认迁移网络vMotion所对应的虚拟交换机
![](https://qiufuqi.github.io/img/hexo/20230410100736.png)
2、找到对应交换机打开点编辑设置，打开安全，在MAC地址更改和伪传输设置成接受。然后点保存即可
![](https://qiufuqi.github.io/img/hexo/20230410100757.png)
