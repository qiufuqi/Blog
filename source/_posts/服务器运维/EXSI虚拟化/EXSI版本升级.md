---
title: EXSI版本升级
date: 2022-10-19
tags:
  - EXSI
  - 升级
categories: 
- 运维
- EXSI
- 升级
keywords: 'Exsi,升级'
cover: https://qiufuqi.github.io/img/hexo/20221019100146.png
abbrlink: exsi_upgrade
comments: false
---

EXSI升级
如果直接5.1升6.7的话，会导致版本不兼容，虚拟机无法使用。选择的方案是5.1升级到6.0版本，然后6.0 升级到6.7版本。
exsi版本	exsi6.5		exsi6.7		exsi7.0
exsi5.5		允许		  不允许		  不允许
exsi6.0		允许		  允许		    不允许
exsi6.5					    允许		    不允许
exsi6.7								          不允许

# 下载升级包
升级包下载地址：https://customerconnect.vmware.com/patch
qiufuqi@foxmail.com   Zxc,./123


# 上传升级包
![](https://qiufuqi.github.io/img/hexo/20231205114912.png)
![](https://qiufuqi.github.io/img/hexo/20231205134255.png)
# 开启SSH
安全配置文件  属性
![](https://qiufuqi.github.io/img/hexo/20231205133801.png)
![](https://qiufuqi.github.io/img/hexo/20231205134342.png)

# 开始升级
每次升级过后，需要reboot重启
``` bash
exsi5.1 升级5.5  查看升级版本包
esxcli software sources profile list -d /vmfs/volumes/datastore1/ESXi550-201312001.zip
Name                              Vendor        Acceptance Level
--------------------------------  ------------  ----------------
ESXi-5.5.0-20131204001-no-tools   VMware, Inc.  PartnerSupported
ESXi-5.5.0-20131204001-standard   VMware, Inc.  PartnerSupported
ESXi-5.5.0-20131201001s-standard  VMware, Inc.  PartnerSupported
ESXi-5.5.0-20131201001s-no-tools  VMware, Inc.  PartnerSupported
esxcli software profile update -d "/vmfs/volumes/datastore1/ESXi550-201312001.zip" -p ESXi-5.5.0-20131204001-standard


加个参数--no-hardware-warning忽略这个警告，看能否升级成功。

exsi5.1 升级5.5  查看升级版本包
esxcli software sources profile list -d /vmfs/volumes/datastore1/ESXi550-201312001.zip
esxcli software profile update -d "/vmfs/volumes/datastore1/ESXi550-201312001.zip" -p ESXi-5.5.0-20131204001-standard


exsi5.5 升级6.0
esxcli software sources profile list -d /vmfs/volumes/datastore1/ESXi600-202002001.zip
esxcli software profile update -d "/vmfs/volumes/datastore1/ESXi600-202002001.zip" -p ESXi-6.0.0-20200204001-standard


exsi6.0 升级6.5
esxcli software sources profile list -d /vmfs/volumes/datastore1/ESXi650-202210001.zip
esxcli software profile update -d "/vmfs/volumes/datastore1/ESXi650-202210001.zip" -p ESXi-6.5.0-20221004001-standard


exsi6.5 升级6.7
esxcli software sources profile list -d /vmfs/volumes/datastore1/ESXi670-202210001.zip
esxcli software profile update -d "/vmfs/volumes/datastore1/ESXi670-202210001.zip" -p ESXi-6.7.0-20221004001-standard


exsi6.7 升级7.0
esxcli software sources profile list -d /vmfs/volumes/datastore1/VMware-ESXi-7.0U3k-21313628-depot.zip
esxcli software profile update -d "/vmfs/volumes/datastore1/VMware-ESXi-7.0U3k-21313628-depot.zip" -p ESXi-7.0U3k-21313628-standard


exsi7.0 升级8
esxcli software sources profile list -d /vmfs/volumes/datastore1/VMware-ESXi-8.0b-21203435-depot.zip
esxcli software profile update -d "/vmfs/volumes/datastore1/VMware-ESXi-8.0b-21203435-depot.zip" -p ESXi-8.0b-21203435-standard
```
