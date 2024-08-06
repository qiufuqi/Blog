---
title: KVM虚拟化搭建
date: 2022-08-15 16:03:45
tags:
  - KVM
categories: 
- 运维
- 虚拟化
keywords: 'Linux,CentOS,KVM,虚拟化'
description: KVM虚拟化搭建
cover: https://qiufuqi.github.io/img/hexo/20231205140759.png
abbrlink: kvm_install
comments: false
top: 89
---

Linux环境部署KVM虚拟化平台，系统版本centos7.6，GUI桌面以及虚拟化支持（安装时）。

## 基本命令

### 查看CPU是否支持VT
``` bash
#重要，安装时需要慎重
egrep '(vmx|svm)' --color=always /proc/cpuinfo
```

### 检查内核模块是否加载
``` bash
lsmod | grep kvm
```

### 查看Selinux状态
如果是启动状态，需要将其关闭
``` bash
sestatus
getenforce

#修改SELINUX=disabled ，然后重启reboot
sudo vim /etc/sysconfig/selinux
```

### 安装KVM
``` bash
yum -y install  kvm libvirt python-virtinst qemu-kvm virt-viewer tunctl bridge-utils avahi dmidecode qemu-kvm-tools virt-manager qemu-img virt-install net-tools libguestfs-tools
```
### 启动libvirt服务
``` bash
systemctl start libvirtd
```

### 设计开机自启
``` bash
systemctl enable libvirtd
```
挂载U盾
https://www.cnblogs.com/kevingrace/p/8016734.html