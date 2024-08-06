---
title: EXSI磁盘回收
date: 2023-07-15
tags:
  - EXSI
  - 磁盘
categories: 
- 运维
- EXSI
- 磁盘回收
keywords: 'Exsi,磁盘回收'
cover: https://qiufuqi.github.io/img/hexo/20221019100146.png
abbrlink: exsi_huishou
comments: false
---

我们在使用ESXI时常常会遇到这么个问题，创建虚拟服务器时使用的磁盘类型为Thin（精简置备）。最初，精简置备的磁盘只使用该磁盘最初所需要的数据存储空间，在使用一段时间后占用磁盘存储空间会变的很大，就算把系统内大文件删除系统内释放了，但是虚拟机的磁盘还是直接占用了之前最大的空间。有没有什么方法可以压缩回收磁盘空间呢？
不防看看下面的方法：

vmkfstools 常用参数选项：
``` bash
-i  指定原磁盘文件名
-d --diskformat 指定目标磁盘的格式（zeroedthick、thin、eagerzeroedthick）
-K --punchzero  回收磁盘空间
```

## 磁盘空间回收
ESXI精简置备类型（Thin）磁盘空间回收,(回收参考)[https://blog.51cto.com/u_13777759/2437396]
**回收磁盘磁盘类型必须为精简置备（thin）回收前最好先备份**
### 打开ESXI服务器SSH,
开启EXSI服务器SSH，开启方法请参考 EXSI开启远程SSH
### 登录EXSI服务器
通过ssh连接ESXI服务器
### 进入虚拟机目录
切换到需要回收的虚拟机目录
``` bash
~ # cd /vmfs/volumes/datastore1
```
### 查看磁盘大小
通过du命令查看该虚拟机磁盘文件大小
``` bash
/vmfs/volumes/55ade938-e958d429-143f-000c29231226/CentOS # du -sh *
```
### 使用vmkfstools命令
通过vmkfstools命令回收空间
``` bash
/vmfs/volumes/55ade938-e958d429-143f-000c29231226/CentOS # vmkfstools -K CentOS.vmdk
```
### 检验大小
``` bash
/vmfs/volumes/55ade938-e958d429-143f-000c29231226/CentOS # du -sh *
```
### 开机测试是否可以正常启动