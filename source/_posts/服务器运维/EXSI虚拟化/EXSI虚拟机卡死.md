---
title: EXSI虚拟机卡死
date: 2023-03-23
tags:
  - EXSI
  - 卡死
  - 重启
categories: 
- 运维
- EXSI
- 重启
keywords: 'Exsi,重启'
cover: https://qiufuqi.github.io/img/hexo/20221019100146.png
abbrlink: exsi_reboot
comments: false
---

在 ESXi 主机上关闭虚拟机电源时，会遇到以下症状：
- 无法关闭 ESXi 托管的虚拟机的电源
- 虚拟机无响应，且无法停止或终止

**使用 SSH 以 root 身份登录到 ESXi**
# 普通关机

获取所有已注册虚拟机的列表，由其 VMID 和显示名称标识
vim-cmd vmsvc/getallvms
``` bash
~ # vim-cmd vmsvc/getallvms
Vmid      Name                        File                           Guest OS        Version   Annotation
1424   sap-router   [datastore1 (4)] sap-router/win2k8x64.vmx   winLonghorn64Guest   vmx-07              
1440   RAS-3.250    [datastore1 (4)] RAS/win2k8x64.vmx          winLonghorn64Guest   vmx-07
```
获取虚拟机当前的状态
vim-cmd vmsvc/power.getstate VMID
``` bash
~ # vim-cmd vmsvc/power.getstate 1440
Retrieved runtime info
Powered on
```
检查受影响的虚拟机上是否有挂起的任务阻止了机器开机
vim-cmd vmsvc/get.tasklist VMID
``` bash
~ # vim-cmd vmsvc/get.tasklist 1440
(ManagedObjectReference) [
  'vim.Task:haTask-2-vim.VirtualMachine.createSnapshot-182550283',
  'vim.Task:haTask-2-vim.VirtualMachine.consolidateDisks-182550274'
]
```
查看任务的更多信息
vim-cmd vimsvc/task_info task_id
``` bash
# vim-cmd vimsvc/task_info haTask-2-vim.VirtualMachine.createSnapshot-182550283
·········
```
任务挂起，需要取消，使用如下命令
``` bash
vim-cmd vimsvc/task_cancel task_id
```
使用命令关闭虚拟机
``` bash
vim-cmd vmsvc/power.shutdown VMID
# 无法关闭时使用以下命令
vim-cmd vmsvc/power.off VMID
```
至此一般能成功关机，如果无法关机则参考以下：强制关机

# 强制关机
## 5.5以上版本
**以下命令适合5.5以上版本**
- 确认虚拟机运行在哪个esxi主机上，使用SSH登陆到该主机（这个应该是esxi运维最基本的操作了，不会自行百度）
- 通过命令找到虚拟机运行的worldID(和进程ID相似，一台虚拟机有一个唯一的ID)

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
记下world id ，执行强制关机命令
``` bash
esxcli vm process kill --type= [soft,hard,force] --world-id= WorldNumber
esxcli vm process kill -t [ soft,hard,force] -w WorldNumber

例如：
esxcli vm process kill --type=force --world-id=68762
esxcli vm process kill -t force -w 68762
```
-t,--type 执行类型
soft： 执行正常关机，调用vmearetool执行关机
hard： 执行立即关机
force：强制断电关机
-w,--world-id
这里指定虚拟机的World ID号了 

## 5.5以下版本
- 运行以下命令获取正在运行的虚拟机的列表（虚拟机由 World ID、UUID、显示名称和 .vmx 配置文件的路径标识）：
- 运行以下命令关闭此列表中某个虚拟机的电源：
``` bash
esxcli vms vm list
esxcli vms vm kill --type= [soft,hard,force] --world-id= WorldNumber
```