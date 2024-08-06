---
title: IBM X3650 M3配置raid5
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
abbrlink: ibm_raid5
comments: false
---

**IBM X3650 M3服务器上RAID配置**

#### 进入Web BIOS
- 进入Web BIOS启动机器，自检过程中会有 <Ctrl> + <H> 的提示，同时按下这两个键，再点击Start，就可以进入WEB BIOS的图形设置界面
![](https://qiufuqi.github.io/img/hexo/20221110164336.png)
![](https://qiufuqi.github.io/img/hexo/20221110164355.png)
- 系统启动后，按F1进入bios进行设置  选择system setting => Adpters and UEFI Drivers => 选择LSI EFI SAS Driver下的PCI的驱动设置
![](https://qiufuqi.github.io/img/hexo/20221110164633.png)
![](https://qiufuqi.github.io/img/hexo/20221110164715.png)
![](https://qiufuqi.github.io/img/hexo/20221110164355.png)

#### 进入设置步骤
进入WebBIOS后点击Configuration Wizard，进入阵列设置向导
![](https://qiufuqi.github.io/img/hexo/20221110164756.png)
有三个选项：
- Clear Configuration（清除配置）：清除已有的配置信息，注意会丢失所有的数据
- New Configuration（全新配置）：清除已有的配置信息，并且全新创建新的配置
- Add Configuration（添加配置）：保留原有配置信息，并且添加新的硬盘到原有的配置中（该配置通常不会引起数据丢失，但该操作有风险，建议先备份数据！）
![](https://qiufuqi.github.io/img/hexo/20221110164814.png)

点Next进入下一步，选择Yes确定
![](https://qiufuqi.github.io/img/hexo/20221110164851.png)

选择手动配置 Manual Configuration，选择Next
![](https://qiufuqi.github.io/img/hexo/20221110164905.png)

左边选择要配置在阵列中的硬盘，然后按Add to Array，从左边的Drivers中选到右边 Drivers Groups中的Drive group 0做RAID 1
![](https://qiufuqi.github.io/img/hexo/20221110164929.png)

然后选择Accept DG，创建新的Drive group 1，再选择左边的硬盘Add to Array到右边的Drive group 1中做JBOD或RAID 0，选好硬盘后，选择 Accept DG后点击Next进入下图界面
![](https://qiufuqi.github.io/img/hexo/20221110164957.png)

点Next进入下图，在左侧的ArrayWithFreeSpace中选中刚才做好的 Disk Groups，按Add to SPAN添加到右侧的span中，然后点Next
![](https://qiufuqi.github.io/img/hexo/20221110165029.png)

进入Virtual Disk配置界面，选好Virtual Disk参数后，点选Accept接受配置，Next
- RAID Level中可以选择要配置的RAID级别
- 右侧的Possible RAID Level中显示可能的RAID级别的磁盘容量，比如示例中三个73G的硬盘配置raid0容量约为200G，而如果配置RAID5容量约为134G
- Select size选项中可以修改Virtual Disk的容量，通常这个值设定为该磁盘组RAID级别的最大容量，注意单位选择GB
![](https://qiufuqi.github.io/img/hexo/20221110165111.png)

如要对Drive group 1做操作，按照对Drive group 0的操作重复即可
在最后的确认画面里点选Accept接受配置，然后会要求确认保存配置和初始化清除数据，就完成了对阵列的配置
![](https://qiufuqi.github.io/img/hexo/20221110165130.png)












