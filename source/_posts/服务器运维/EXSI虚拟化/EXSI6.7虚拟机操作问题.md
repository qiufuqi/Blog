---
title: EXSI6.7虚拟机操作问题
date: 2022-12-09
tags:
  - EXSI
  - 开机/关机
categories: 
- 运维
- EXSI
- 开机/关机
keywords: 'Exsi,虚拟机,开机/关机'
cover: https://qiufuqi.github.io/img/hexo/20221209095615.png
abbrlink: exsi_caozuo
comments: false
---

**解决esxi虚拟机无法关机、重置以及开机操作**
当exsi管理平台虚拟机无法关机，重置以及开机操作时，一般有两种方法操作

## 命令行管理
当出现无法操作时候，控制台、包括API都无法使用了，我们需要使用vmware底层命令来设置虚拟机的状态

进入esxi的web管理，开启esxi的shell登录。
![](https://qiufuqi.github.io/img/hexo/20221209094512.png)

1.确认虚拟机运行在哪个esxi主机上，使用SSH登陆到该主机
2.通过命令找到虚拟机运行的worldID(和进程ID相似，一台虚拟机有一个唯一的ID)
esxcli vm process list
``` bash
[root@localhost:~] esxcli vm process list
RH236_3.1
   World ID: 68762
   Process ID: 0
   VMX Cartel ID: 68761
   UUID: 56 4d 70 ad 43 f1 5e 25-8a 12 a0 e2 18 2e 38 ec
   Display Name: RH236_3.1
   Config File: /vmfs/volumes/5cce4874-40c5c88e-a836-003048fc4ffe/RH236_3.1/RH236_3.1.vmx
 
Server 2016 Vcenter
   World ID: 68982
   Process ID: 0
   VMX Cartel ID: 68981
   UUID: 56 4d d5 80 19 b2 b9 a1-ea 0b cd 89 a6 05 e9 48
   Display Name: Server 2016 Vcenter
   Config File: /vmfs/volumes/5cce65ea-ab8c59fa-82c7-003048fc4ffe/Server 2016 Vcenter/Server 2016 Vcenter.vmx
```
记下world id 
![](https://qiufuqi.github.io/img/hexo/20221209094659.png)
通过命令强制结束掉虚拟机
``` bash
esxcli vm process kill --type= [soft,hard,force] --world-id= WorldNumber
或者
esxcli vm process kill -t [ soft,hard,force] -w WorldNumber

esxcli vm process kill --type=force --world-id=68762
esxcli vm process kill -t force -w 68762
```
t,--type 执行类型
soft： 执行正常关机，调用vmearetool执行关机
hard： 执行立即关机
force：强制断电关机
-w,--world-id

这里指定虚拟机的World ID号了 
## 重启物理机
去机房重启机器即可，但是会影响机器上的其他虚拟机




