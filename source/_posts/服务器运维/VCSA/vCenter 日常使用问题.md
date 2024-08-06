---
title: vCenter日常使用问题
date: 2023-2-24
tags:
  - EXSI
  - vCenter
categories: 
- 运维
- EXSI
- vCenter
keywords: 'Exsi,vCenter'
cover: https://qiufuqi.github.io/img/hexo/20221207140120.png
abbrlink: exsi_vCenter_problem
comments: false
---

**vCenter日常使用问题**

### 无法开机问题
[参考](https://www.virten.net/2017/08/vcsa-6-5-broken-filesystem-welcome-to-emergency-mode/)
版本：VCSA 6.5 Broken Filesystem - "Welcome to Emergency Mode"
``` bash
Welcome to emergency mode! After logging in, type "journalctl -xb" to view system logs, "systemctl reboot" to reboot, "systemctl default" or ^D to try again to boot into default mode.
Give root password for maintenance
(or press Control-D to continue):
```
![](https://qiufuqi.github.io/img/hexo/20230224141346.png)
(Shift + Page Up)查看日志
![](https://qiufuqi.github.io/img/hexo/20230224141507.png)
问题提示为log出现问题：[FAILED] Failed to start Filesystem Check on **/dev/log_vg/log.**

输入密码进入系统，使用df -h 和 cat /etc/fstab 可以发现 /storage/log 系统在df -h 中丢失了
![](https://qiufuqi.github.io/img/hexo/20230224141639.png)

使用如下命令修复，全部yes
``` bash
# e2fsck -y /dev/log_vg/log
```
![](https://qiufuqi.github.io/img/hexo/20230224141740.png)

重启解决问题
``` bash
# reboot -f
```

### 磁盘空间不足
一般就是VMDK的5和13，分别是10GB的/storage/log和50GB的/storage/archive
建议直接扩容到50GB和250GB
使用命令vpxd_servicecfg storage lvm autogrow来自动扩展
服务起不来就重启service-control --start --all











