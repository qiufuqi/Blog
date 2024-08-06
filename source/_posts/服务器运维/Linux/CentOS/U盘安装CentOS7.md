---
title: U盘安装CentOS7
date: 2022-08-14 16:59:50
tags:
  - Linux
  - CentOS
  - 系统
categories: 
- 运维
- 系统
keywords: 'Linux,CentOS,U盘'
description: U盘安装CentOS7
cover: https://qiufuqi.github.io/img/hexo/20231205135714.png
abbrlink: centos_u
comments: false
---

CentOS7.6 物理机安装（服务器单独部署CentOS环境）
前置条件：服务器 && U盘启动盘（提前导入CentOS7系统）

## 安装步骤：

1.把U盘插到电脑上

2.设置开机U盘启动 ps：机器不一样设置也不一样具体请百度，我的是按F12可选择U盘。

3.选择U盘后跳转到下图界面
![](https://qiufuqi.github.io/img/hexo/20231205135345.png)

4.按下键盘TAB键将以下信息修改
``` bash
vmlinuz initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 rd.live.check quiet
# 修改为
vmlinuz initrd=initrd.img linux dd quiet
```
![](https://qiufuqi.github.io/img/hexo/20231205133550.png)
![](https://qiufuqi.github.io/img/hexo/20231205115020.png)

5.查看U盘启动盘的名称比如：sda，sdb，sdc  ps：label一列会显示Centos7等字样的
![](https://qiufuqi.github.io/img/hexo/20231205133641.png)
6.重启后到第三步界面按下TAB键

7.根据第5步查询到的启动盘名称进行修改
``` bash
vmlinuz initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 rd.live.check quiet 
# 修改为  ps：sdb4就是你看到的启动盘名称
vmlinuz initrd=initrd.img inst.stage2=hd:/dev/sdb4 quiet   
```
8.之后等待安装到图形界面

后续安装可参考[CentOS系统安装](/centos_install)