---
title: EXSI6.7磁盘扩容
date: 2022-09-20
tags:
  - EXSI
  - 磁盘
categories: 
- 运维
- EXSI
- 磁盘
keywords: 'Exsi,磁盘'
cover: https://qiufuqi.github.io/img/hexo/20220922143249.png
abbrlink: exsi_cipan
comments: false
---

注意：扩容磁盘的方式分为 [添加磁盘]、[扩展磁盘] ;
# 添加磁盘-在线增加
进入EXSI管理平台，看到原来的硬盘只有一块，点击 添加硬盘-新标准硬盘
![](https://qiufuqi.github.io/img/hexo/20221026134042.png)
## 确认添加状态
登陆机器，查看磁盘，发现多了一块sdb
PS:在工作中，有时我们会遇到如下一种情况，就是在对Vmware虚拟机外部添加了一块磁盘
然后我们登录到Linux系统中却发现不了新的磁盘，解决方法一般有如下两种
1、重启Linux操作系统（对于生产环境,就不是很推荐了）
2、通过在Linux操作系统中，刷新扫描scsi host 设备
重启不提了，介绍下第二种方法:
刷新扫描scsi设备和，如果没有出结果，可以继续host3，host4，直到操作系统中能够识别出新添加的磁盘
``` bash
[root@qq-5201351 ~]# echo "- - -" > /sys/class/scsi_host/host0/scan
[root@qq-5201351 ~]# echo "- - -" > /sys/class/scsi_host/host1/scan
[root@qq-5201351 ~]# echo "- - -" > /sys/class/scsi_host/host2/scan
```
刷新后一般也就出现了

``` bash
# 新增硬盘前
[root@master-node ~]# lsblk
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda               8:0    0  100G  0 disk 
├─sda1            8:1    0    1G  0 part /boot
└─sda2            8:2    0   99G  0 part 
  ├─centos-root 253:0    0   50G  0 lvm  /
  ├─centos-swap 253:1    0  7.9G  0 lvm  [SWAP]
  └─centos-home 253:2    0 41.1G  0 lvm  /home
sr0              11:0    1 1024M  0 rom  

# 新增硬盘后
[root@master-node ~]# lsblk
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda               8:0    0  100G  0 disk 
├─sda1            8:1    0    1G  0 part /boot
└─sda2            8:2    0   99G  0 part 
  ├─centos-root 253:0    0   50G  0 lvm  /
  ├─centos-swap 253:1    0  7.9G  0 lvm  [SWAP]
  └─centos-home 253:2    0 41.1G  0 lvm  /home
sdb               8:16   0   50G  0 disk 
sr0              11:0    1 1024M  0 rom
```
或者使用fdisk确认磁盘空间是否已经扩展
``` bash
[root@master-node ~]# fdisk -l
Disk /dev/sda: 107.4 GB, 107374182400 bytes, 209715200 sectors
·········

Disk /dev/sdb: 53.7 GB, 53687091200 bytes, 104857600 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```
## 扩展分区
``` bash
[root@master-node ~]# fdisk /dev/sdb
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table
Building a new DOS disklabel with disk identifier 0x7f92d23a.

Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p
Partition number (1-4, default 1): 
First sector (2048-104857599, default 2048): 
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-104857599, default 104857599): 
Using default value 104857599
Partition 1 of type Linux and of size 50 GiB is set

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks
```
使用lsblk或者fdisk命令 再次查看磁盘，发现多了个sdb1
``` bash
[root@master-node ~]# lsblk
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda               8:0    0  100G  0 disk 
├─sda1            8:1    0    1G  0 part /boot
└─sda2            8:2    0   99G  0 part 
  ├─centos-root 253:0    0   50G  0 lvm  /
  ├─centos-swap 253:1    0  7.9G  0 lvm  [SWAP]
  └─centos-home 253:2    0 41.1G  0 lvm  /home
sdb               8:16   0   50G  0 disk 
└─sdb1            8:17   0   50G  0 part 
sr0              11:0    1 1024M  0 rom

# 或者fdisk命令
[root@master-node ~]# fdisk -l
Disk /dev/sda: 107.4 GB, 107374182400 bytes, 209715200 sectors
·········

Disk /dev/sdb: 53.7 GB, 53687091200 bytes, 104857600 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x7f92d23a

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048   104857599    52427776   83  Linux
```
## 硬盘格式化
格式化硬盘，这里使用xfs格式，建议使用和系统一样的文件格式
可以看到/分区 【/dev/mapper/centos-root】 使用的是xfs 的文件系统
``` bash
[root@master-node ~]# blkid
/dev/sda1: UUID="80258da0-bb03-42f4-abaf-580ad558de6f" TYPE="xfs" 
/dev/sda2: UUID="kWBKkc-iH5s-m1wm-56s6-3md0-eY9F-77RFpW" TYPE="LVM2_member" 
/dev/mapper/centos-root: UUID="bda3808d-ebe7-47e8-b093-47be3bad0c20" TYPE="xfs" 
/dev/mapper/centos-swap: UUID="e1df6176-5508-49a7-8ba2-9ff25248298c" TYPE="swap" 
/dev/mapper/centos-home: UUID="30c12979-6fc4-47c0-afd8-825eca9bb0f7" TYPE="xfs"
```
把/dev/sdb1 格式化成 xfs文件系统
``` bash
[root@master-node ~]# mkfs.xfs /dev/sdb1
meta-data=/dev/sdb1              isize=512    agcount=4, agsize=3276736 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=13106944, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=6399, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

# 看到/dev/sdb1 的文件系统为 xfs
[root@master-node ~]# blkid
/dev/sda1: UUID="80258da0-bb03-42f4-abaf-580ad558de6f" TYPE="xfs" 
/dev/sda2: UUID="kWBKkc-iH5s-m1wm-56s6-3md0-eY9F-77RFpW" TYPE="LVM2_member" 
/dev/mapper/centos-root: UUID="bda3808d-ebe7-47e8-b093-47be3bad0c20" TYPE="xfs" 
/dev/mapper/centos-swap: UUID="e1df6176-5508-49a7-8ba2-9ff25248298c" TYPE="swap" 
/dev/mapper/centos-home: UUID="30c12979-6fc4-47c0-afd8-825eca9bb0f7" TYPE="xfs" 
/dev/sdb1: UUID="cb155145-5841-473c-980e-7571bd346a01" TYPE="xfs"
```
## Lvm扩容
创建pv物理卷
``` bash
[root@master-node ~]# pvcreate /dev/sdb1 
WARNING: xfs signature detected on /dev/sdb1 at offset 0. Wipe it? [y/n]: y
  Wiping xfs signature on /dev/sdb1.
  Physical volume "/dev/sdb1" successfully created.
```
查看pv物理卷，查看vg卷组名称为centos（或者使用vgdisplay）
``` bash
[root@master-node ~]# pvs
  PV         VG     Fmt  Attr PSize   PFree  
  /dev/sda2  centos lvm2 a--  <99.00g   4.00m
  /dev/sdb1         lvm2 ---  <50.00g <50.00g

[admin@localhost ~]$ vgs
  VG     #PV #LV #SN Attr   VSize   VFree
  centos   1   3   0 wz--n- <59.00g    0 

# 使用其他命令，同样vg name为centos
[root@master-node ~]# vgdisplay
  --- Volume group ---
  VG Name               centos
  ·········
  VG Size               <99.00 GiB
  PE Size               4.00 MiB
  Total PE              25343
  Alloc PE / Size       25342 / 98.99 GiB
  Free  PE / Size       1 / 4.00 MiB
  VG UUID               yTqRKf-eSrx-KgEe-D34S-U9vX-lhGf-UfnW2p
```
 向vg卷组添加 pv物理卷sdb1
 ``` bash
[root@master-node ~]# vgextend centos /dev/sdb1
  Volume group "centos" successfully extended
```
可发现空闲PE，容量大约50G
``` bash
[root@master-node ~]# vgs
  VG     #PV #LV #SN Attr   VSize   VFree 
  centos   2   3   0 wz--n- 148.99g 50.00g
[root@master-node ~]# vgdisplay
  --- Volume group ---
  VG Name               centos
  ·········
  VG Size               148.99 GiB
  PE Size               4.00 MiB
  Total PE              38142
  Alloc PE / Size       25342 / 98.99 GiB
  Free  PE / Size       12800 / 50.00 GiB
  VG UUID               yTqRKf-eSrx-KgEe-D34S-U9vX-lhGf-UfnW2p
 ```
## 扩展lv
通过lvs或者lvdisplay 可以看到分区的lv名称为 root
``` bash
[root@master-node ~]# lvs
  LV   VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home centos -wi-ao---- <41.12g                                                    
  root centos -wi-ao----  50.00g                                                    
  swap centos -wi-ao----  <7.88g

[root@master-node ~]# lvdisplay
  --- Logical volume ---
  LV Path                /dev/centos/swap
  ······
  --- Logical volume ---
  LV Path                /dev/centos/home
  ······
  --- Logical volume ---
  LV Path                /dev/centos/root
  ······
```
我们把新扩展的50G全部添加到centos-root中
``` bash
[root@master-node ~]# lvextend /dev/mapper/centos-root /dev/sdb1
  Size of logical volume centos/root changed from 50.00 GiB (12800 extents) to <100.00 GiB (25599 extents).
  Logical volume centos/root successfully resized.
```
使用lvs或者lsblk可以看到空间已经增加上了，但是df-h并没有变化 可选择重启或者**刷新文件系统**
``` bash
[root@master-node ~]# lvs
  LV   VG     Attr       LSize    Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home centos -wi-ao----  <41.12g                                                    
  root centos -wi-ao---- <100.00g                                                    
  swap centos -wi-ao----   <7.88g
[root@master-node ~]# lsblk
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda               8:0    0  100G  0 disk 
├─sda1            8:1    0    1G  0 part /boot
└─sda2            8:2    0   99G  0 part 
  ├─centos-root 253:0    0  100G  0 lvm  /
  ├─centos-swap 253:1    0  7.9G  0 lvm  [SWAP]
  └─centos-home 253:2    0 41.1G  0 lvm  /home
sdb               8:16   0   50G  0 disk 
└─sdb1            8:17   0   50G  0 part 
  └─centos-root 253:0    0  100G  0 lvm  /
sr0              11:0    1 1024M  0 rom 
[root@master-node ~]# df -klh
Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   50G  1.4G   49G   3% /
devtmpfs                 3.9G     0  3.9G   0% /dev
tmpfs                    3.9G     0  3.9G   0% /dev/shm
tmpfs                    3.9G  8.9M  3.9G   1% /run
tmpfs                    3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda1               1014M  145M  870M  15% /boot
/dev/mapper/centos-home   42G   33M   42G   1% /home
tmpfs                    783M     0  783M   0% /run/user/0
```
## 刷新文件系统
``` bash
[root@master-node ~]# df -T
Filesystem              Type     1K-blocks    Used Available Use% Mounted on
/dev/mapper/centos-root xfs       52403200 1417504  50985696   3% /
devtmpfs                devtmpfs   3992520       0   3992520   0% /dev
tmpfs                   tmpfs      4004596       0   4004596   0% /dev/shm
tmpfs                   tmpfs      4004596    9052   3995544   1% /run
tmpfs                   tmpfs      4004596       0   4004596   0% /sys/fs/cgroup
/dev/sda1               xfs        1038336  148404    889932  15% /boot
/dev/mapper/centos-home xfs       43093444   32992  43060452   1% /home
tmpfs                   tmpfs       800920       0    800920   0% /run/user/0
[root@master-node ~]# xfs_growfs /dev/mapper/centos-root
meta-data=/dev/mapper/centos-root isize=512    agcount=4, agsize=3276800 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=13107200, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=6400, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 13107200 to 26213376

# 新增的50G磁盘已经有效
[root@master-node ~]# df -h
Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root  100G  1.4G   99G   2% /
devtmpfs                 3.9G     0  3.9G   0% /dev
tmpfs                    3.9G     0  3.9G   0% /dev/shm
tmpfs                    3.9G  8.9M  3.9G   1% /run
tmpfs                    3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda1               1014M  145M  870M  15% /boot
/dev/mapper/centos-home   42G   33M   42G   1% /home
tmpfs                    783M     0  783M   0% /run/user/0
```



# 扩展磁盘-关机增加
扩展磁盘需要在此虚拟机停止的状态下进行，同时扩展的数字是扩展后的预期大小，比如之前是60G，希望扩展200G，那么我们应该输入200G。这里我们以扩展磁盘的方式进行。
![](https://qiufuqi.github.io/img/hexo/20220920141306.png)

## 确认添加状态
扩展后，重新启动linux，使用df -kh命令发现磁盘目录大小没有变化
``` bash
[admin@localhost ~]$ df -h
Filesystem Size Used Avail Use% Mounted on
/dev/mapper/centos-root 36G 1.6G 35G 5% /
devtmpfs 3.9G 0 3.9G 0% /dev
tmpfs 3.9G 0 3.9G 0% /dev/shm
tmpfs 3.9G 8.9M 3.9G 1% /run
tmpfs 3.9G 0 3.9G 0% /sys/fs/cgroup
/dev/sda1 1014M 189M 826M 19% /boot
/dev/mapper/centos-home 18G 33M 18G 1% /home
tmpfs 783M 0 783M 0% /run/user/0
tmpfs 783M 0 783M 0% /run/user/1001
```
使用fdisk确认磁盘空间是否已经扩展
``` bash
[admin@localhost ~]$ fdisk -l

Disk /dev/sda: 214.7 GB, 214748364800 bytes, 419430400 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x0005871c

Device Boot Start End Blocks Id System
/dev/sda1 * 2048 2099199 1048576 83 Linux
/dev/sda2 2099200 125829119 61864960 8e Linux LVM

Disk /dev/mapper/centos-root: 38.2 GB, 38235275264 bytes, 74678272 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

Disk /dev/mapper/centos-swap: 6442 MB, 6442450944 bytes, 12582912 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

Disk /dev/mapper/centos-home: 18.7 GB, 18668847104 bytes, 36462592 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```
可以看到“Disk /dev/sda: 214.7 GB”，已经扩展了140G空间。

## 扩展分区
``` bash
[admin@localhost ~]$ fdisk /dev/sda
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): n
Partition type:
   p   primary (2 primary, 0 extended, 2 free)
   e   extended
Select (default p): p
Partition number (3,4, default 3): 
First sector (125829120-419430399, default 125829120): 
Using default value 125829120
Last sector, +sectors or +size{K,M,G} (125829120-419430399, default 419430399): 
Using default value 419430399
Partition 3 of type Linux and of size 140 GiB is set

Command (m for help): t
Partition number (1-3, default 3): 
Hex code (type L to list all codes): L

 0  Empty           24  NEC DOS         81  Minix / old Lin bf  Solaris        
 1  FAT12           27  Hidden NTFS Win 82  Linux swap / So c1  DRDOS/sec (FAT-
 2  XENIX root      39  Plan 9          83  Linux           c4  DRDOS/sec (FAT-
 3  XENIX usr       3c  PartitionMagic  84  OS/2 hidden C:  c6  DRDOS/sec (FAT-
 4  FAT16 <32M      40  Venix 80286     85  Linux extended  c7  Syrinx         
 5  Extended        41  PPC PReP Boot   86  NTFS volume set da  Non-FS data    
 6  FAT16           42  SFS             87  NTFS volume set db  CP/M / CTOS / .
 7  HPFS/NTFS/exFAT 4d  QNX4.x          88  Linux plaintext de  Dell Utility   
 8  AIX             4e  QNX4.x 2nd part 8e  Linux LVM       df  BootIt         
 9  AIX bootable    4f  QNX4.x 3rd part 93  Amoeba          e1  DOS access     
 a  OS/2 Boot Manag 50  OnTrack DM      94  Amoeba BBT      e3  DOS R/O        
 b  W95 FAT32       51  OnTrack DM6 Aux 9f  BSD/OS          e4  SpeedStor      
 c  W95 FAT32 (LBA) 52  CP/M            a0  IBM Thinkpad hi eb  BeOS fs        
 e  W95 FAT16 (LBA) 53  OnTrack DM6 Aux a5  FreeBSD         ee  GPT            
 f  W95 Extd (LBA) 54  OnTrackDM6      a6  OpenBSD         ef  EFI (FAT-12/16/
10  OPUS            55  EZ-Drive        a7  NeXTSTEP        f0  Linux/PA-RISC b
11  Hidden FAT12    56  Golden Bow      a8  Darwin UFS      f1  SpeedStor      
12  Compaq diagnost 5c  Priam Edisk     a9  NetBSD          f4  SpeedStor      
14  Hidden FAT16 <3 61  SpeedStor       ab  Darwin boot     f2  DOS secondary  
16  Hidden FAT16    63  GNU HURD or Sys af  HFS / HFS+      fb  VMware VMFS    
17  Hidden HPFS/NTF 64  Novell Netware  b7  BSDI fs         fc  VMware VMKCORE 
18  AST SmartSleep  65  Novell Netware  b8  BSDI swap       fd  Linux raid auto
1b  Hidden W95 FAT3 70  DiskSecure Mult bb  Boot Wizard hid fe  LANstep        
1c  Hidden W95 FAT3 75  PC/IX           be  Solaris boot    ff  BBT            
1e  Hidden W95 FAT1 80  Old Minix      
Hex code (type L to list all codes): 8e     
Changed type of partition 'Linux' to 'Linux LVM'

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.

WARNING: Re-reading the partition table failed with error 16: Device or resource busy.
The kernel still uses the old table. The new table will be used at
the next reboot or after you run partprobe(8) or kpartx(8)
Syncing disks.
```
## 加载分区表
方法一：（推荐）
``` bash
[root@localhost ~]# partprobe
```
执行 partprobe命令用于将磁盘分区表变化信息通知内核，并请求操作系统重新加载分区表，此方法可以不用重启系统；
方法二：重启系统
``` bash
[root@localhost ~]# reboot
```

## 分区确认
通过fdisk可以看到已经添加了/dev/sda3
``` bash
[admin@localhost ~]$ fdisk -l    

Disk /dev/sda: 214.7 GB, 214748364800 bytes, 419430400 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x0005871c

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     2099199     1048576   83  Linux
/dev/sda2         2099200   125829119    61864960   8e  Linux LVM
/dev/sda3       125829120   419430399   146800640   8e  Linux LVM

Disk /dev/mapper/centos-root: 38.2 GB, 38235275264 bytes, 74678272 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

Disk /dev/mapper/centos-swap: 6442 MB, 6442450944 bytes, 12582912 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

Disk /dev/mapper/centos-home: 18.7 GB, 18668847104 bytes, 36462592 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```
## LVM扩容
创建pv物理卷
``` bash
[admin@localhost ~]$ pvcreate /dev/sda3
Physical volume "/dev/sda3" successfully created.
```
查看pv物理卷
``` bash
[root@master-node ~]# pvs
  PV         VG     Fmt  Attr PSize   PFree  
  /dev/sda2  centos lvm2 a--  <99.00g   4.00m
  /dev/sda3         lvm2 ---  <50.00g <50.00g
```

使用vgs查看
``` bash
[admin@localhost ~]$ vgs
  VG     #PV #LV #SN Attr   VSize   VFree
  centos   1   3   0 wz--n- <59.00g    0 
```
把sda3加入到LVM组中 注意：centos 是vg组名称，请根据具体情况填写
``` bash
[admin@localhost ~]$ vgextend centos /dev/sda3 
  Volume group "centos" successfully extended
```
查看发现磁盘空余大约140G
``` bash
[admin@localhost ~]$ vgs
  VG     #PV #LV #SN Attr   VSize   VFree   
  centos   2   3   0 wz--n- 198.99g <140.00g
```
## 扩展lv
我们把新扩展的100G全部添加到centos-root中
``` bash
[admin@localhost ~]$ lvextend /dev/mapper/centos-root  /dev/sda3 
  Size of logical volume centos/root changed from <35.61 GiB (9116 extents) to <175.61 GiB (44955 extents).
  Logical volume centos/root successfully resized.
```
使用lvs可以看到 centos-root 已经是140G了，但是…请继续往下看
``` bash
[admin@localhost ~]$ lvs
  LV   VG     Attr       LSize    Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home centos -wi-ao----  <17.39g                                                    
  root centos -wi-ao---- <175.61g                                                    
  swap centos -wi-ao----    6.00g 
```
使用df -kh查看，空间并没有变化，look down… 可选择重启或者**刷新文件系统**
``` bash
[admin@localhost ~]$ df -khl
Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   36G  1.6G   35G   5% /
devtmpfs                 3.9G     0  3.9G   0% /dev
tmpfs                    3.9G     0  3.9G   0% /dev/shm
tmpfs                    3.9G  8.9M  3.9G   1% /run
tmpfs                    3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda1               1014M  189M  826M  19% /boot
/dev/mapper/centos-home   18G   33M   18G   1% /home
tmpfs                    783M     0  783M   0% /run/user/1001
```
## 刷新文件系统
``` bash
[admin@localhost ~]$ df -T
Filesystem              Type     1K-blocks    Used Available Use% Mounted on
/dev/mapper/centos-root xfs       37320904 1616800  35704104   5% /
devtmpfs                devtmpfs   3992828       0   3992828   0% /dev
tmpfs                   tmpfs      4004744       0   4004744   0% /dev/shm
tmpfs                   tmpfs      4004744    9036   3995708   1% /run
tmpfs                   tmpfs      4004744       0   4004744   0% /sys/fs/cgroup
/dev/sda1               xfs        1038336  192612    845724  19% /boot
/dev/mapper/centos-home xfs       18221056   33020  18188036   1% /home
tmpfs                   tmpfs       800952       0    800952   0% /run/user/1001
[admin@localhost ~]$ xfs_growfs /dev/mapper/centos-root 
meta-data=/dev/mapper/centos-root isize=512    agcount=4, agsize=2333696 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=9334784, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=4558, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 9334784 to 46033920

# 再次确认df状态, 添加的100G空间已经有效
[admin@localhost ~]$ df -kh
Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root  176G  1.6G  175G   1% /
devtmpfs                 3.9G     0  3.9G   0% /dev
tmpfs                    3.9G     0  3.9G   0% /dev/shm
tmpfs                    3.9G  8.9M  3.9G   1% /run
tmpfs                    3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda1               1014M  189M  826M  19% /boot
/dev/mapper/centos-home   18G   33M   18G   1% /home
tmpfs                    783M     0  783M   0% /run/user/1001
```

# 转移磁盘
从CentOS7默认安装的/home中转移空间到根目录
``` bash
mkdir /backup  
mv /home/* /backup/  
umount /home  
lvremove /dev/centos/home  
lvcreate -L 50G -n home centos    # home目录留下50G
mkfs -t xfs /dev/centos/home  
mount /dev/centos/home /home
mv /backup/* /home/  
lvextend -L +42G /dev/centos/root  # 将新增42G到root
xfs_growfs /dev/centos/root  
rm -rf /backup
```