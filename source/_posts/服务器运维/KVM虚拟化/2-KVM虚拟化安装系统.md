---
title: KVM虚拟化安装系统
date: 2022-08-16 16:19:45
tags:
  - KVM
  - System
  - Clone
categories: 
- 运维
- 虚拟化
- 系统
keywords: 'Linux,CentOS,KVM,系统,System'
description: KVM虚拟化安装系统
cover: https://qiufuqi.github.io/img/hexo/20231205140957.png
abbrlink: kvm_system_install
comments: false
top: 88
---

Linux环境部署KVM虚拟化平台，系统版本centos7.6，GUI桌面以及虚拟化支持（安装时）。

## 简单介绍
一般来说，KVM安装虚拟机有多种方式。

## 操作页面安装

### 打开管理工具
在host机的终端，执行 # virt-manager 命令。稍等一会儿，会打开kvm，如下：
``` bash
[root@node1 ~]# virt-manager
[root@node1 ~]# libGL error: No matching fbConfigs or visuals found
libGL error: failed to load driver: swrast
```
![](https://qiufuqi.github.io/img/hexo/20231205141742.png)

### 点击新建虚拟机
点击 文件 => 新建虚拟机 => 选择本地安装介质 => 使用ISO镜像 => 点击浏览
![](https://qiufuqi.github.io/img/hexo/20231205141814.png)
![](https://qiufuqi.github.io/img/hexo/20231205141703.png)
![](https://qiufuqi.github.io/img/hexo/20220818110828.png)
### 选择镜像
可以提前创建文件夹并上传镜像 => 设置虚拟机配置 => 设置虚拟机存放位置 => 点击管理 => 创建虚拟磁盘
![](https://qiufuqi.github.io/img/hexo/20220818111100.png)
![](https://qiufuqi.github.io/img/hexo/20220818111241.png)
![](https://qiufuqi.github.io/img/hexo/20220818111445.png)
![](https://qiufuqi.github.io/img/hexo/20220818111628.png)

### 创建虚拟机
创建虚拟机 => 名称 && 网络（网桥单独介绍） 自定义配置要选择 => 更改协议如图 => 开始安装
![](https://qiufuqi.github.io/img/hexo/20220818111804.png)
![](https://qiufuqi.github.io/img/hexo/20220818112017.png)

### 系统安装
系统安装流程参考之前文档

### 安装问题

#### 鼠标无法捕捉
可点击灯泡，选择重新引导安装
``` bash
1、关闭kvm虚拟机

2、在/etc/libvirt/qemu下找到对应的xml配置文件
在<devices>标签下添加
<input type='tablet' bus='usb'/>

# 保存文件后 执行下面的命令
virsh define /etc/libvirt/qemu/xx.xml # xx为你的配置文件名称

3、启动kvm虚拟机

```
#### 键盘无法输入
在detail中找到display vnc, keymap选择 en-us 后重启虚拟机，虚拟机启动后再次输入
![](https://qiufuqi.github.io/img/hexo/20231205141316.png)

#### 切换vlan
点击灯泡，切换不通vlan
![](https://qiufuqi.github.io/img/hexo/20220818151059.png)


## 命令行安装

## 克隆方式安装
[克隆方式请参考](/kvm_system_clone)