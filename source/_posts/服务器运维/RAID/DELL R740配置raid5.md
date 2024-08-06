---
title: DELL R740配置raid5
date: 2022-11-10
tags:
  - Linux
  - DELL
  - RAID
categories: 
- 运维
- RAID
keywords: 'Linux,RAID,DELL'
cover: https://qiufuqi.github.io/img/hexo/20221110162906.png
abbrlink: dell740_raid5
comments: false
---

**Dell R740服务器配置RAID5+1**
RAID5+1，一块盘做热备，最少3块盘，部分dell机型已不支持BIOS做raid，推荐使用dell普通模式

# 普通模式raid

开机点击F2，选择Device Settings
![](https://qiufuqi.github.io/img/hexo/20230714084708.png)
![](https://qiufuqi.github.io/img/hexo/20230714084800.png)

选择第一个RAID控制器：RAID Controller in SL 3: Dell PERC H755 Front Configuration Utility
![](https://qiufuqi.github.io/img/hexo/20230714084859.png)

进入主菜单，有不同的选项，选择第一个Configuration Management
![](https://qiufuqi.github.io/img/hexo/20230714085022.png)
(Physical Disk Management查看物理磁盘)

进入管理页面，首先选择:Clear Configuration 清除配置(dell 会自动安装raid，需要手动清除换成自己所需要的)
![](https://qiufuqi.github.io/img/hexo/20230714145012.png)

创建虚拟化磁盘：Create Virtual Disk，进入创建页面，选择raid level，并选择磁盘
![](https://qiufuqi.github.io/img/hexo/20230714145130.png)
![](https://qiufuqi.github.io/img/hexo/20230714145220.png)

选择创建虚拟化磁盘，至此创建完成，可进行查看
![](https://qiufuqi.github.io/img/hexo/20230714145252.png)

![](https://qiufuqi.github.io/img/hexo/20230714145327.png)
![](https://qiufuqi.github.io/img/hexo/20230714145340.png)


# BIOS做raid
## F2到设置页面
开机点击F2选择第一个选项进入到BIOS
![](https://qiufuqi.github.io/img/hexo/20221110163059.png)
## 选择Boot Settings
![](https://qiufuqi.github.io/img/hexo/20221110163112.png)
## 选择BIOS
选择BIOS，然后按Esc，保存配置重启服务器
![](https://qiufuqi.github.io/img/hexo/20221110163124.png)
## 进入阵列卡
服务器重启后，重复按Ctrl+R进入阵列卡，如下图所示：
![](https://qiufuqi.github.io/img/hexo/20221110163135.png)
## 点击F2
选择倒数第二个选项 
![](https://qiufuqi.github.io/img/hexo/20221110163151.png)
## 空格磁盘
空格全选所有磁盘，Tab切换到OK确认
![](https://qiufuqi.github.io/img/hexo/20221110163200.png)
## 点击F2
点击F2进入第一个选项：
![](https://qiufuqi.github.io/img/hexo/20221110163211.png)
## 选择RAID
如果有RAID-6就直接全选所有硬盘然后选择等级为RAID-6
![](https://qiufuqi.github.io/img/hexo/20221110163223.png)
## 点击OK按钮
OK后退出如下图：
![](https://qiufuqi.github.io/img/hexo/20221110163317.png)
## 光标移至红框处按F2
![](https://qiufuqi.github.io/img/hexo/20221110163329.png)
## 选择Manage Ded, HS
![](https://qiufuqi.github.io/img/hexo/20221110163348.png)
## 选中剩余硬盘：
选中剩余的一个一块硬盘然后点击OK：
![](https://qiufuqi.github.io/img/hexo/20221110163357.png)
## 点击F2
选中刚才配置的硬盘点击F2，快速初始化硬盘
![](https://qiufuqi.github.io/img/hexo/20221110163407.png)
## RAID 5+1磁盘阵列已配置完成
如下所示RAID 5+1磁盘阵列已配置完成，重启服务器即可使用U盘安装系统：
![](https://qiufuqi.github.io/img/hexo/20221110163420.png)
## 重启系统安装
重启进入到Boot Manager选择One-shot BIOS Boot Menu,选择对应插入的U盘就可进入到系统安装环节。
![](https://qiufuqi.github.io/img/hexo/20221110163431.png)
![](https://qiufuqi.github.io/img/hexo/20221110163440.png)