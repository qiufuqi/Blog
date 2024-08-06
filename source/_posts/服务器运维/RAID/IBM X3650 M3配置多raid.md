---
title: IBM X3650 M3配置多raid
date: 2022-11-10
tags:
  - Linux
  - IBM
  - RAID
categories: 
- 运维
- RAID
keywords: 'Linux,RAID,IBM'
cover: https://qiufuqi.github.io/img/hexo/20221110165300.png
abbrlink: ibm_raid_more
comments: false
---

**IBM X3650 M3服务器上多RAID配置**
[配置参考](https://blog.51cto.com/u_15233520/5226348)

物理机，两块系统盘做raid1，其余盘每块硬盘做raid0，4块SATA和2块SSD   
![](https://qiufuqi.github.io/img/hexo/20221207170745.png)
![](https://qiufuqi.github.io/img/hexo/20221207170756.png)
![](https://qiufuqi.github.io/img/hexo/20221207170820.png)
![](https://qiufuqi.github.io/img/hexo/20221207170834.png)

调整 cache写策略为write through     
![](https://qiufuqi.github.io/img/hexo/20221207170847.png)
![](https://qiufuqi.github.io/img/hexo/20221207170856.png)

全部硬盘 raid做完： 
![](https://qiufuqi.github.io/img/hexo/20221207170910.png)




