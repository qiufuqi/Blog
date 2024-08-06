---
title: Linux无法进入系统
date: 2022-12-09
tags:
  - Linux
  - CentOS
  - 无法启动
categories: 
- 运维
- 无法启动
keywords: 'Linux,CentOS,无法启动'
cover: https://qiufuqi.github.io/img/hexo/20221209095706.png
abbrlink: centos_errorstart
comments: false
---

**开机进入了紧急救援模式，提示Failed to start system check on /dev/disk/by-uuid/**
提示登录后查看以下服务
``` bash
systemctl status "systemd-fsck@dev-disk-by\\x2duuid-c90f924a\\x2d0d00\\x2d4a22\\x2db52c\\x2d5d87f98d54d6.service""

# 输出以下命令
error read block 37748820 （Input/Output error）....
fsck exited with status code 4
run fsck manually
```
查看一下fsck的错误类型
``` bash
man fsck
...
              0      No errors
              1      Filesystem errors corrected
              2      System should be rebooted
              4      Filesystem errors left uncorrected
              8      Operational error
              16     Usage or syntax error
              32     Checking canceled by user request
              128    Shared-library error
...
```
跑一遍fsck,然后重启
``` bash
fsck -y /dev/sda6
```