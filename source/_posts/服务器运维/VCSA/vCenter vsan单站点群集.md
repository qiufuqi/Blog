---
title: vSAN单站点群集
date: 2022-12-07
tags:
  - EXSI
  - vCenter
  - vSAN
categories: 
- 运维
- EXSI
- vCenter
keywords: 'Exsi,vCenter,vSAN'
cover: https://qiufuqi.github.io/img/hexo/20221207140120.png
abbrlink: exsi_vCenter_vSAN
comments: false
---


**VMware vSAN 6.7 搭建单站点vSAN群集**
VMware vSAN 是作为 ESXi 管理程序的一部分本机运行的分布式软件层。
vSAN 可汇总主机群集的本地或直接连接容量设备，并创建在 vSAN 群集的所有主机之间共享的单个存储池。
vSAN 支持 HA、vMotion 和 DRS 等需要共享存储的 VMware 功能，但它无需外部共享存储，并且简化了存储配置和虚拟机置备活动。
注：如果主机向 vSAN 数据存储提供其本地存储设备，则必须至少提供一个闪存缓存设备和一个容量设备；容量设备也称为数据磁盘，此类主机上的设备将构成一个或多个磁盘组；每个磁盘组包含一个闪存缓存设备，以及一个或多个用于持久存储的容量设备；每个主机都可配置为使用多个磁盘组。

在单站点vSAN群集的环境下，我们最少需要三台服务器来搭建一个群集，一份数据以镜像方式分别保存在其中两台服务器A和B上，这样就能够提供高可靠的数据保护。
在单站点vSAN群集中，见证的角色是随机分配的，虚拟机对象的见证组件也是随机保存在群集中的某一台主机上（RAID1的存储策略下）。

[部署参考地址](https://www.qedev.com/tag/VMwarevSAN)
# 安装准备
前期准备：
- 搭建好vCenter vSphere Client管理平台，[搭建参考](/exsi_vCenter_install)。
- 三台EXSI6.7主机，系统盘+闪存盘+容量盘，[部署参考](/ibm_raid_more)。
- 交换机，用于内部通信

三台EXSI6.7主机ip：
10.11.8.10
10.11.8.13
10.11.8.14

# 配置vSphere群集

## 添加主机
在vCenter中新建数据中心，把3台主机都加入vCenter，并进入维护模式
![](https://qiufuqi.github.io/img/hexo/20221207142926.png)
## 开启SSH
选中主机 => 配置 => 系统 => 服务 => SSH ，启动即可，也可编辑启动策略
![](https://qiufuqi.github.io/img/hexo/20221207143140.png)
## 标记磁盘
选中主机 => 配置 => 存储 => 存储适配器 => 选中磁盘 => 设备 ，将其中一块盘标记为闪存盘
![](https://qiufuqi.github.io/img/hexo/20221207143430.png)
## 开启vMotion
选中主机 => 配置 => 网络 => VMkernel适配器 => 选中网卡 => 编辑，开启vMotion
![](https://qiufuqi.github.io/img/hexo/20221212135102.png)
![](https://qiufuqi.github.io/img/hexo/20221212135116.png)
## 新建集群
在数据中心下新建集群test_vsan_cluster，并将3台主机拖入集群
![](https://qiufuqi.github.io/img/hexo/20221207143845.png)
![](https://qiufuqi.github.io/img/hexo/20221207143933.png)
## 调整存储名称
本步骤可不做，但是为了方便管理，以ip最后数字命名
![](https://qiufuqi.github.io/img/hexo/20221207144045.png)

# 配置启用vSAN
## 新建分布式交换机
新建分布式交换机，建立vSAN存储网络
![](https://qiufuqi.github.io/img/hexo/20221207144235.png)
输入分布式交换机名称test_DSwitch_vsan
![](https://qiufuqi.github.io/img/hexo/20221207144352.png)
选择版本，3台主机exsi的版本
![](https://qiufuqi.github.io/img/hexo/20221207144431.png)
上行链路数目为1，每台主机都有1条网卡来连接vSAN的。顺便创建默认的端口组。
![](https://qiufuqi.github.io/img/hexo/20221207144609.png)
完成分布式交换机创建
![](https://qiufuqi.github.io/img/hexo/20221207144643.png)
## 交换机添加主机配置链路
添加和管理主机，添加主机
![](https://qiufuqi.github.io/img/hexo/20221207145105.png)
+新主机
![](https://qiufuqi.github.io/img/hexo/20221207145157.png)
![](https://qiufuqi.github.io/img/hexo/20221207145224.png)
分配上行链路，选中网卡（用于内部通信）
如果机器相同，且连接网卡相同，则选择将此上行链路分配到所有主机
![](https://qiufuqi.github.io/img/hexo/20221207145605.png)
暂时不管理网络适配器，下一步直至完成
![](https://qiufuqi.github.io/img/hexo/20221207145902.png)
到每台ESXi主机虚拟网络处可以看到此时分布式交换机的拓扑，选中主机 => 配置 => 网络 => 虚拟交换机
![](https://qiufuqi.github.io/img/hexo/20221207150012.png)

为分布式端口组添加VMkernel网卡，配置分布式交换机的左端
![](https://qiufuqi.github.io/img/hexo/20221207150221.png)
选择所有主机
![](https://qiufuqi.github.io/img/hexo/20221207150252.png)
务必选中vSAN服务
![](https://qiufuqi.github.io/img/hexo/20221207150337.png)

配置ip地址，图片中172.16.0.0/24网段，用于内部通信（交换机端设置放行）
![](https://qiufuqi.github.io/img/hexo/20221207150413.png)
配置完成
![](https://qiufuqi.github.io/img/hexo/20221207150525.png)
返回ESXi主机查看网络，可以看到分布式交换机的左端（端口组）已经配置。
![](https://qiufuqi.github.io/img/hexo/20221207150556.png)

至此，分布式交换机配置完成。

此时可登录exsi主机，确保各个主机间正常通信。
![](https://qiufuqi.github.io/img/hexo/20221207150729.png)

## 创建磁盘组
启用vSAN前，请保证要用来做磁盘组的磁盘（缓存盘和数据盘）不包含任何分区信息。未消耗
![](https://qiufuqi.github.io/img/hexo/20221207150840.png)
开启vsan服务，选中集群 => 配置 => vSAN
![](https://qiufuqi.github.io/img/hexo/20221207150924.png)

选择单站点群集，如果缓存盘为SSD，固态盘，可开启去重和压缩
![](https://qiufuqi.github.io/img/hexo/20221207151046.png)
声明磁盘，（二次实验，已经声明过了，所以此处不显示，参考第二章图片，区分容量层和缓存层）
如果磁盘被使用过，需要初始化硬盘，[参考](/exsi_vCenter_vSAN_problem)，解决
![](https://qiufuqi.github.io/img/hexo/20221208100202.png)


故障域在此先不声明，点击下一步完成
![](https://qiufuqi.github.io/img/hexo/20221207151451.png)
正在配置vSAN群集，等待任务执行完毕即可
![](https://qiufuqi.github.io/img/hexo/20221207151552.png)
将所有主机退出维护模式，到群集的数据存储处查看 vsanDatastore 的容量
![](https://qiufuqi.github.io/img/hexo/20221207151955.png)
查看vSAN的运行状况，保证其不出现红色的警告
![](https://qiufuqi.github.io/img/hexo/20221207152045.png)

**至此，一个单站点的vSAN群集搭建成功。**
创建虚拟机时可选择存储位置为vsandatastore中，具体来说是将虚拟机的对象（主页和hard disk）以网络RAID1的方式存放在多台ESXi主机上的，实现多副本，避免单点故障，某一台主机故障（进入维护模式或者掉电），虚拟机仍可运行。