---
title: vCenter重置密码问题
date: 2022-12-16
tags:
  - EXSI
  - vCenter
categories: 
- 运维
- EXSI
- vCenter
keywords: 'Exsi,vCenter,重置密码'
cover: https://qiufuqi.github.io/img/hexo/20220922143047.png
abbrlink: exsi_vCenter_passwd
comments: false
---

**Vmware vCenter 6.7重置密码**

[参考地址](https://blog.csdn.net/yzqtcc/article/details/124577270)

vCenter root 密码和SSO登入密码忘记，及VC出现（log Disk Exhaustion on vc）日志盘满的报错

进行以下操作时，做个**快照**或者其他可以回退的方法
有时候web界面键盘输入就会有各种问题（也是为避免误操作），建议下载VMRC或者用Workstations对VC进行命令操作。

# 重置root密码
## 重启vCenter
在看到 Photon OS 启动屏幕，按 e 键进入 GNU GRUB 编辑菜单。到Linux这一行的末尾输入 rw init=/bin/bash 然后按F10继续引导。
![](https://qiufuqi.github.io/img/hexo/20221216165146.png)
## 运行重置命令
``` bash
mount -o remount,rw /
passwd
```
![](https://qiufuqi.github.io/img/hexo/20221216165229.png)
![](https://qiufuqi.github.io/img/hexo/20221216165239.png)
## 重启vCenter
输入reboot -f 重启VC

# 重置administrator密码
通过root登录vcenter系统
## 执行重置操作命令
运行 /usr/lib/vmware-vmdir/bin/vdcadmintool
![](https://qiufuqi.github.io/img/hexo/20221216165842.png)
## 输入UPN账户
提示输入帐户 UPN （就是登入VC web界面的账户）然后会出现一串负复杂的随机的密码。
![](https://qiufuqi.github.io/img/hexo/20221216165931.png)









