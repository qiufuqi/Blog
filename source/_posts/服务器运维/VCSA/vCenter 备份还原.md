---
title: vCenter 备份还原
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
abbrlink: exsi_vCenter_recover
comments: false
---

**vCenter备份还原**

# 登陆到VC控制台
端口：5480
找到备份-立即备份
前提需要找一个windows服务器创建一个共享目录，本章介绍使用smb方式进行备份和恢复，其他方式类似。
![](https://qiufuqi.github.io/img/hexo/20230224153604.png)
# 填写备份位置
chown ftp_vcenter:ftp ./test

![](https://qiufuqi.github.io/img/hexo/20230224153633.png)
![](https://qiufuqi.github.io/img/hexo/20230224153644.png)
正在开始备份
![](https://qiufuqi.github.io/img/hexo/20230224153704.png)
![](https://qiufuqi.github.io/img/hexo/20230224153710.png)
# 恢复VCSA7.0
找到安装VCSA7.0的系统镜像
![](https://qiufuqi.github.io/img/hexo/20230224153725.png)
这里选择还原
![](https://qiufuqi.github.io/img/hexo/20230224153746.png)
![](https://qiufuqi.github.io/img/hexo/20230224153757.png)
输入备份服务器的账号和密码
FTP://10.11.7.95/
![](https://qiufuqi.github.io/img/hexo/20230224153812.png)
输入部署在那台ESXI7.0服务器
![](https://qiufuqi.github.io/img/hexo/20230224153827.png)
输入VCSA7.0密码
![](https://qiufuqi.github.io/img/hexo/20230224153847.png)
选择部署大小
![](https://qiufuqi.github.io/img/hexo/20230224153904.png)
选择数据存储
![](https://qiufuqi.github.io/img/hexo/20230224153925.png)
配置VCSA7.0的网络信息
![](https://qiufuqi.github.io/img/hexo/20230224153938.png)

然后点击下一步即将完成第一阶段，和安装VCSA7.0步骤几乎相同，请参考安装说明。
