---
title: EXSI6.7挂载存储
date: 2024-01-10
tags:
  - EXSI
  - 存储
categories: 
- 运维
- EXSI
- 存储
keywords: 'Exsi,存储'
cover: https://qiufuqi.github.io/img/hexo/20221019100146.png
abbrlink: exsi_cunchu
comments: false
---

EXSI挂载存储，存储设备浪潮，直连交换机

浪潮存储默认设置
管理IP：192.168.70.100
用户名：superuser
密码：Passw0rd!

# EXSI端设置
## 网络
### 物理网卡
确保物理网卡，有一条链路是给连接存储使用 （可选择直连存储，本文是连接在交换机上）
![](https://qiufuqi.github.io/img/hexo/20240110093037.png)
### 虚拟交换机
新建虚拟交换机，明确物理网卡链路
![](https://qiufuqi.github.io/img/hexo/20240110093346.png)
![](https://qiufuqi.github.io/img/hexo/20240110093400.png)
### 端口组
新建端口组
![](https://qiufuqi.github.io/img/hexo/20240110093453.png)
![](https://qiufuqi.github.io/img/hexo/20240110093508.png)
### VMkernel网卡
添加VMkernel网卡，IPv4 设置IP，此处的IP在存储里会使用
![](https://qiufuqi.github.io/img/hexo/20240110093609.png)
![](https://qiufuqi.github.io/img/hexo/20240110093652.png)

## 存储
### 设备扫描
设备-重新扫描
![](https://qiufuqi.github.io/img/hexo/20240110093839.png)
### 适配器
适配器-开启iSCSI，并配置：网络端口绑定，设置静态目标和动态目标，记录下名称和别名，存储里使用
![](https://qiufuqi.github.io/img/hexo/20240110094823.png)
### 新建数据存储
数据存储-新建数据存储
![](https://qiufuqi.github.io/img/hexo/20240110093851.png)

# 存储端设置
存储本身连接一条网线做管理口，再连接一条网线用于和使用存储设备的机器通信(通过交换机)
## 配置网络
设置-网络-管理IP地址
![](https://qiufuqi.github.io/img/hexo/20240110095944.png)
## 配置以太网端口
设置-网络-以太网端口； 和虚拟化出VMkernel网卡设置的ip在同一网段
![](https://qiufuqi.github.io/img/hexo/20240110100101.png)

## 新建池
新增等硬盘，创建新的即可
![](https://qiufuqi.github.io/img/hexo/20240110094205.png)
![](https://qiufuqi.github.io/img/hexo/20240110094253.png)
## 新建卷
创建卷，并命名，可创建多个卷
![](https://qiufuqi.github.io/img/hexo/20240110095324.png)
![](https://qiufuqi.github.io/img/hexo/20240110095200.png)
## 主机
添加主机，拷贝iSCSI端口（exsi主机，适配器中拷贝）
![](https://qiufuqi.github.io/img/hexo/20240110095434.png)
![](https://qiufuqi.github.io/img/hexo/20240110095714.png)








