---
title: EXSI磁盘格式
date: 2023-07-15
tags:
  - EXSI
  - 磁盘
categories: 
- 运维
- EXSI
- 磁盘格式
keywords: 'Exsi,磁盘格式'
cover: https://qiufuqi.github.io/img/hexo/20221019100146.png
abbrlink: exsi_format
comments: false
---
Esxi里虚拟机磁盘类型厚置备改精简置备

# 本机修改
## 启动SSH
在ESXi的Web页面的【主机】页打开ssh功能。
![](https://qiufuqi.github.io/img/hexo/20230715144309.png)
## 登录ESXI CLI
使用ssh工具连接虚拟机，可以使用PuTTY
## 进入虚拟机目录
进入存放虚拟机的目录，然后进入要转换的虚拟机的目录
## 查看文件
输入“ls -lh”，可以看到有一个很小的vmdk和一个带-flat的体积较大的vmdk，实际上数据是存储在大的那个里，小的是信息。
![](https://qiufuqi.github.io/img/hexo/20230715144441.png)
## 转换格式
输入“vmkfstools -i 原.vmdk -d thin 新.vmdk”开始转换，中间-d thin的参数是关键
![](https://qiufuqi.github.io/img/hexo/20230715144622.png)
## 再次查看
再次输入“ls -lh”就能看到多了一个centos-original_0_new.vmdk和一个centos-original_0_new-flat.vmdk
![](https://qiufuqi.github.io/img/hexo/20230715144800.png)
## 变更文件
将新建的vmdk改为原来的vmdk的名字
``` bash
mv centos-original_0.vmdk centos-original_0.vmdk.bak
mv centos-original_0-flat.vmdk centos-original_0-flat.vmdk.bak

mv centos-original_0_new-flat.vmdk oentos-original_0-flat.vmdk
mv centos-original_0_new.vmdk centos-original_0.vmdk
```
![](https://qiufuqi.github.io/img/hexo/20230715144811.png)
## 修改文件
输入“vi centos-original_0.vmdk”编辑它。将红框这一行的文件名改成“centos-original_0-flat.vmdk”。保存。
![](https://qiufuqi.github.io/img/hexo/20230715145304.png)
执行命令vi centos-original.vmx 搜索vmdk，将关联的vmdk文件名改为新的vmdk（不带flat）文件名
![](https://qiufuqi.github.io/img/hexo/20230715145328.png)

## 重新注册该虚拟机
先在ESXi的【虚拟机】页面取消注册这个虚拟机
![](https://qiufuqi.github.io/img/hexo/20230715145401.png)
重新注册虚拟机。
![](https://qiufuqi.github.io/img/hexo/20230715145417.png)

# vCenter修改
在虚拟机迁移时选择修改磁盘模式