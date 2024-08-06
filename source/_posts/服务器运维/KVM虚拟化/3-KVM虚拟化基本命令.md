---
title: KVM虚拟化基本命令
date: 2022-08-15 16:38:45
tags:
  - KVM
  - Bash
categories: 
- 运维
- 虚拟化
- 命令
keywords: 'Linux,CentOS,KVM,虚拟化,Bash'
description: KVM虚拟化基本命令
cover: https://qiufuqi.github.io/img/hexo/20231205140834.png
abbrlink: kvm_bash
comments: false
top: 87
---

KVM虚拟化常用命令

## KVM的基本管理

### 查看KVM虚拟机配置文件
``` bash
#Kvm虚拟机默认配置文件位置
[root@node1 qemu]# pwd
/etc/libvirt/qemu
[root@node1 qemu]# ll
总用量 16
drwxr-xr-x. 2 root root   42 5月   2 15:42 autostart
drwx------. 3 root root   42 4月  28 2021 networks
-rw-------. 1 root root 4259 4月  27 16:54 test11.xml
-rw-------. 1 root root 4259 4月  27 14:14 test12.xml
```

### 启动与关闭
``` bash
#显示正在运行的虚拟机
[root@node1 qemu]# virsh list
 Id    名称                         状态
----------------------------------------------------
 1     test11                         running
 2     test12                         running
#显示所有虚拟机
[root@node1 qemu]# virsh list --all
 Id    名称                         状态
----------------------------------------------------
 1     test11                         running
 2     test12                         running
```

### 启动虚拟机
``` bash
[root@node1 qemu]# virsh start test11
```

### 关闭虚拟机
``` bash
[root@node1 qemu]# virsh shutdown test11
#或者对应id
[root@node1 qemu]# virsh shutdown 1
```

### 强制关闭虚拟机 
``` bash
[root@node1 qemu]# virsh destroy test11
```

### 移除虚拟机 
``` bash
[root@node1 qemu]# virsh undefine test11
```

### 设置虚拟机开机启动 
``` bash
[root@node1 qemu]# virsh autostart test11
[root@node1 qemu]# virsh autostart --disable test11
```

默认情况下virsh工具不能对linux虚拟机进行关机操作
linux操作系统需要开启与启动acpid服务。在安装KVM linux虚拟机必须配置此服务。
``` bash
# yum -y install acpid
# /etc/init.d/acpid start
```

### 通过配置文件启动虚拟机
``` bash
[root@node1 qemu]# virsh create /etc/libvirt/qemu/test11.xml
```
### 挂起，恢复 virsh命令
``` bash
#挂起服务器
[root@node1 qemu]# virsh suspend test11

#恢复服务器
[root@node1 qemu]# virsh resume test11
```

### 重命名虚拟机
``` bash
#停止虚拟机
[root@node1 qemu]# virsh shutdown test11

#导出虚拟机的配置文件
[root@node1 qemu]# pwd
/etc/libvirt/qemu
[root@node1 qemu-img]# virsh dumpxml test11 > test-test11.xml

#更改配置文件
[root@node1 qemu]# sed -i 's/test11/test-test11/g' test-test11.xml
#注 可以不用更改镜像名

#移除原有的虚拟机
[root@node1 qemu]# virsh undefine test11

#加载新建的虚拟机
[root@node1 qemu]# virsh define test-test11.xml

#启动虚拟机
[root@node1 qemu]# virsh start test-test11
```


## 虚拟机快照操作

### 转换磁盘镜像文件格式为qcow2
``` bash
[root@node1 ~]# virsh shutdown test-test11
[root@node1 qemu-img]# qemu-img convert -f raw test11.raw -O qcow2 test11.raw.qcow2
#快照一定需要qcow2格式才行 
#我这边从新建立一个虚拟机以qcow2
```

### 创建快照
``` bash
[root@kvm qemu-img]# virsh snapshot-create test11
```
### 查看快照
``` bash
[root@kvm qemu-img]# virsh snapshot-list test11
 Name                 Creation Time             State
------------------------------------------------------------
 1660579077           2022-08-15 23:57:57 +0800 shutoff
```
### 恢复快照
``` bash
[root@kvm qemu-img]# virsh snapshot-revert test11  1660579077
```

### 删除快照
``` bash
[root@kvm qemu-img]# virsh snapshot-delete test11 1660579077
```