---
title: Linux文件系统
date: 2023-5-30
tags:
  - Linux
  - CentOS
  - Shell
  - 文件系统
categories: 
- 运维
- Linux
- 文件系统
keywords: 'Linux,CentOS,文件系统'
cover: https://qiufuqi.github.io/img/hexo/20230418114913.png
abbrlink: centos_file_system
comments: false
---

# Linux文件系统
79页
Linux使用树形文件存储结构，在磁盘上存储文件的时候，使用的是目录加文件的形式。文件系统+虚拟文件系统VFS
Linux支持不同的文件系统，包括ext2，ext3，ext4，zfs，iso9660，vfat，msdos，smbfs，nfs等。
- 超级块：文件系统的总体信息，是文件系统的核心 （存在多个超级块）
- i节点：存储所有与文件有关的元数据，所有者，权限等属性数据
- 数据块：真是存放文件数据的部分
- 目录快：文件名和文件在目录中的位置

## 磁盘分区、创建文件系统、挂载
磁盘分区分为两类：主分区和扩展分区
### 创建文件系统
fdisk 创建分区，格式化文件系统
mkfs -t ext3 /dev/sdb1 或者 mkfs.ext3 /dev/sdb1 两个命令hi一样的。
### 磁盘挂载
mount 挂载设备后文件系统的分区才能使用
mount /dev/sdb1 newdisk   # 挂载到newdisk目录下
mount                     # 没有参数的mount会显示所有挂载
### 启动自动挂载
/etc/fstab 开机自动挂载磁盘，将/dev/sdb1挂载到/root/newDisk，文件系统时ext3
echo "/dev/sdb1 /root/newDisk ext3 defaults 0 0" >>/etc/fstab
### 磁盘检验
fsck检测磁盘，badblocks检测磁盘物理坏道
当磁盘出现逻辑错误时，可以使用fsck来尝试修复（突然掉电）,fsck检查磁盘时，需要磁盘是为挂载的状态，否则可能造成文件损坏
当系统的跟文件系统需要fsck时，只能重启，系统会自动检测。
用法：fsck -t TYPE /DEVICE/PATH
``` bash
# 解除挂载
umount /dev/sdb1 
unmount /root/newDisk

fsck -t ext3 /dev/sdb1
```
badblocks主要用来检测磁盘的物理坏道，一般怀疑磁盘有坏道时才使用
``` bash
badblocks -v /dev/sdb1
```

## 制作逻辑卷
### 创建物理卷
pvcreate将分区创建为PV，可使用pvsacn查看
``` bash
pvcreate /dev/sdb1
```
pvdisplay可更详细的显示PV的使用状态
``` bash
pvdisplay
```
### 创建卷组
有了PV就可以创建卷组了,vgcreate来创建卷组
``` bash
vgcreate VG_NAME DEVICE1 DEVICE2
```
vgdisplay显示当前系统所有的VG

### 扩容卷组
使用中需要扩到VG_NAME，可以使用vgextend随时扩大VG的容量
``` bash
vgextend VG_NAME DEVICE1 DEVICE2

#示例 将/dev/sdc3 做出PV，再扩容到VG中
pvcreate /dev/sdc3
vgextend vg_ame /dev/sdc3
```
### 创建逻辑卷
有了卷组，就可以创建逻辑卷：lvcreate
``` bash
# -L指定逻辑卷大小50G， -n逻辑卷名字， VG_NAME指定从什么卷组中分配空间
lvcreate -L SIZE -n LV_NAME VG_NAME
```
lvdisplay显示当前系统所有的逻辑卷

### 创建文件系统并挂载
逻辑卷创建好后需要创建文件系统，挂载后爱能使用
``` bash
mkfs.ext3 /dev/卷组名/逻辑卷名
mount /dev/卷组名/逻辑卷名 /root/文件夹
```

## 硬链接和软连接
### 硬链接
通过索引节点来进行链接。多个文件名指向同一个索引节点是被允许的，这个就是硬链接。
硬链接允许一个文件拥有多个有效路径名，删除其中一个连接不影响其他，只有删除最后一个链接时，资源才会被释放。
ln 文件 快捷方式
- 不允许给目录创建硬链接
- 只有同一个文件系统的文件之间才能创建链接。

``` bash
# hard01的inode为70 第一列inode，第三列源文件关联数
[root@localhost hard]# ls -li
总用量 0
70 -rw-r--r--. 1 root root 0 5月  30 16:27 hard01
[root@localhost hard]# ln hard01 hard_likn
[root@localhost hard]# ls -li       # 源文件关联数 2 了
总用量 0
70 -rw-r--r--. 2 root root 0 5月  30 16:27 hard01
70 -rw-r--r--. 2 root root 0 5月  30 16:27 hard_likn
```
### 软链接
软链接时一个包含了另一个文件路径名的文件，可以指向任意文件或目录，也可以跨不同的文件系统。
删除软链接不会删除其指向的源文件，如果删除了源文件则软链接会出现断链。
ln -s 文件 快捷方式
``` bash
[root@localhost hard]# ln -s hard01 hh
[root@localhost hard]# ls -li   # 拥有自己的inode
总用量 0
70 -rw-r--r--. 2 root root 0 5月  30 16:27 hard01
70 -rw-r--r--. 2 root root 0 5月  30 16:27 hard_likn
69 lrwxrwxrwx. 1 root root 6 5月  30 16:42 hh -> hard01
```
