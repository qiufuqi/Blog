---
title: vCenter HA高可用
date: 2024-1-10
tags:
  - EXSI
  - vCenter
  - HA
categories: 
- 运维
- EXSI
- vCenter
keywords: 'Exsi,vCenter'
cover: https://qiufuqi.github.io/img/hexo/20221207140120.png
abbrlink: exsi_vCenter_HA
comments: false
---

**VMware vCenter HA 配置** [文章参考](https://mp.weixin.qq.com/s/hQOw9pDwn-Y-3HOc1BR2HA)
vCenter Server是整个vSphere环境的管理中心，其高可用性和备份通常采用其他方法，以下是一些常见的做法：
- vCenter HA： 使用vCenter HA可以实现vCenter Server的高可用性。它创建一个三节点的vCenter Server群集，确保在主节点故障时能够快速切换到备份节点。
- 定期备份：使用vSphere Data Protection（VDP）或其他备份解决方案，定期备份vCenter Server的数据和配置。这样，在发生故障时，可以还原到最近的备份点。
- 数据库备份：如果vCenter Server使用外部数据库（如SQL Server或Oracle），定期备份数据库以确保vCenter Server的数据完整性。
  
主要介绍vCenter HA的部署。必要条件
- 必须使用VCSA方式部署vCenter。
- 必须6.5以上版本。    
- 部署前需要打开VCSA SSH。
- 事先规划好vCenter HA网络。

## 添加端口组
为每台ESXi主机添加一个HA端口组。（物理适配器在生产环境中建议添加两个做冗余，我这里实验环境就省略了）
主机-配置-虚拟交换机-添加网络-标准交换机的虚拟机端口组
![](https://qiufuqi.github.io/img/hexo/20240110105603.png)
![](https://qiufuqi.github.io/img/hexo/20240110105643.png)

## 部署vCenter HA
1.点击vCenter主机，点击“配置”，点击“vCenter HA”，点击“设置VCENTER HA”。
![](https://qiufuqi.github.io/img/hexo/20240110110059.png)
2.选择主动节点的HA网络，点击浏览。
![](https://qiufuqi.github.io/img/hexo/20240110110125.png)

3.选择创建的“HA”端口组，点击确定。
![](https://qiufuqi.github.io/img/hexo/20240110110141.png)
4.设置被动节点，点击“编辑”。
![](https://qiufuqi.github.io/img/hexo/20240110110201.png)

5.指定名称和位置。
![](https://qiufuqi.github.io/img/hexo/20240110110217.png)
6.选择计算资源，因为第一台VC是放在192.168.99.40上的，所以这里选择192.168.99.121。
![](https://qiufuqi.github.io/img/hexo/20240110110309.png)
7.选择存储位置，根据自己实际情况选择点击下一步。  
![](https://qiufuqi.github.io/img/hexo/20240110110322.png)

8.选择网络，管理网络选择“VM Network”，HA网络选择“HA”，点击下一步。
![](https://qiufuqi.github.io/img/hexo/20240110110339.png)
9.确认配置信息，点击完成。
![](https://qiufuqi.github.io/img/hexo/20240110110357.png)
10.设置见证节点，重复上述5-10步骤,注意选择的计算资源应不与另外两台同属一台ESXi主机。
![](https://qiufuqi.github.io/img/hexo/20240110110457.png)

11.配置完成后，点击下一页。 
![](https://qiufuqi.github.io/img/hexo/20240110110526.png)
12.配置HA网络，网关可以不设置，点击完成。（注意HA网络与管理网络不能同一网段）
![](https://qiufuqi.github.io/img/hexo/20240110110544.png)
13.等待设置完成（过程比较慢大概1小时左右）
![](https://qiufuqi.github.io/img/hexo/20240110110607.png)

14.部署完成后可以看到三台vCenter正在运行分别是：活动、被动、见证。这时候被动节点的管理IP和活动节点一致，但被动为灰色。当活动节点出现故障的时候会自动切换到被动节点。
![](https://qiufuqi.github.io/img/hexo/20240110110629.png)

到此vCenter HA的部署就完成了。

## 验证HA
在当前状态下，我这里直接将“主VC”关机。    
![](https://qiufuqi.github.io/img/hexo/20240110110654.png)

这时候vSphere web界面就已经打不开了。
![](https://qiufuqi.github.io/img/hexo/20240110110707.png)

等了大概8分钟左右，就已经切换过来了，这时候vSphere能正常打开了。
![](https://qiufuqi.github.io/img/hexo/20240110110718.png)

登录上去后发现，会有报错提示“vCenter HA 集群已丢失一个节点”，并且此时主被动节点也自动更换了。
![](https://qiufuqi.github.io/img/hexo/20240110110730.png)

此时我再次将原主节点开机，当原主节点开机后这里还是显示为“被动节点”，但是管理IP地址为灰色。
![](https://qiufuqi.github.io/img/hexo/20240110110749.png)


vCenter HA通过创建三节点的vCenter Server群集，确保vCenter Server在面临硬件故障、软件故障或其他问题时能够快速、自动地切换到备份节点，提高整个vSphere环境的可用性和稳定性。但需要注意的是HA网络不能与管理网络处于同一网段，主备节点切换时间大概在8分钟左右。