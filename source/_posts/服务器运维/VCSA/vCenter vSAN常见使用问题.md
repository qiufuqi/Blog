---
title: vSAN常见使用问题
date: 2022-12-07
tags:
  - EXSI
  - vCenter
  - vSAN
categories: 
- 运维
- EXSI
- vCenter
keywords: 'Exsi,vCenter,vSAN'
cover: https://qiufuqi.github.io/img/hexo/20221207140120.png
abbrlink: exsi_vCenter_vSAN_problem
comments: false
---

vCenter常见问题归类


# 许可证过期
当我们进入系统时，上方会有个明显的提示：清单中包含许可证已过期或即将过期的 vCenter Server 系统。从官方下载的都是申请60天试用的，那么就意味着60天后会过期。
运行许可证生成软件
![](https://qiufuqi.github.io/img/hexo/20221205135047.png)

进入分配许可 “管理您的许可证”——“许可证”——“添加新许可” 输入许可证秘钥
![](https://qiufuqi.github.io/img/hexo/20220921174415.png)

编辑许可证名称 点击完成
![](https://qiufuqi.github.io/img/hexo/20220921174453.png)

许可证添加成功之后，信息如下从灰色!可以得知，其实该许可证还没分配的，所以上方的许可证即将过期的提示还在
进入分配许可证 资产——选择主机——选择分配许可证
![](https://qiufuqi.github.io/img/hexo/20220921174533.png)

分配许可证
选择刚才上面新配置的许可证，选中之后可以看到下面分配验证的提示：许可证分配有效
![](https://qiufuqi.github.io/img/hexo/20220921174557.png)

分配许可证成功之后校验
![](https://qiufuqi.github.io/img/hexo/20220921174637.png)
刷新之后，上方明显：‘清单中包含许可证已过期或即将过期的 vCenter Server 系统’的提示已消失
![](https://qiufuqi.github.io/img/hexo/20220921174656.png)

vSAN许可证过期
选中集群 => 配置 => 配置 => 许可 => 分配许可证
![](https://qiufuqi.github.io/img/hexo/20221207153953.png)
![](https://qiufuqi.github.io/img/hexo/20221207154059.png)

# vSAN 相关配置
HA高可用
其运行机制是监控群集中的ESXi主机及虚拟机，通过配置合适的策略，当群集中的ESXi主机或虚拟机发生故障，可以自动到其他的ESXi主机上进行重新启动，最大限度保证重要服务不中断。
选中集群 => 配置 => 配置 => 快速入门   vSphere HA
![](https://qiufuqi.github.io/img/hexo/20221207154253.png)
 
DRS 动态资源平衡
vSphere DRS 可不间断地平衡资源池内的计算容量，以提供物理基础架构所不能提供的性能、可扩展性和可用性。
选中集群 => 配置 => 配置 => 快速入门   vSphere DRS
![](https://qiufuqi.github.io/img/hexo/20221207154253.png)

故障域
vSAN故障域可以隔离ESXi主机。我们在同一个vSAN群集中，可以把服务器A和B定义在一个故障域中，服务器C、D和E定义在另外一个故障域中，那么vSAN存放虚拟机主机时，不会把组件存放在同一个故障域中的主机上，当某一个故障域出现问题，服务仍然能提供。
选中集群 => 配置 => vSAN => 故障域
![](https://qiufuqi.github.io/img/hexo/20221207162723.png)

# vSAN 解散群集
每个节点删除vsan用于内部通信端口（或者分布式交换机 => 配置 => 拓扑）
![](https://qiufuqi.github.io/img/hexo/20221208101037.png)

分布式交换机删除各个节点 ，最后删除 分布式交换机
![](https://qiufuqi.github.io/img/hexo/20221208101200.png)

选中集群 => 配置 => 配置 => 快速入门，群集基础设置全部取消
![](https://qiufuqi.github.io/img/hexo/20221208101257.png)

每台主机进入维护模式，删除主机

SSH命令行 每个节点执行 退出集群 以及磁盘初始化操作
退出集群：
1、查看VSAN集群状态：esxcli vsan cluster get
2、退出VSAN集群“esxcli vsan cluster leave
初始化磁盘
1、查看缓存磁盘ID和容量磁盘ID：esxcli vsan storage list
2、移除磁盘：esxcli vsan storage remove -u 52d6bb30-6b39-df6f-3543-a4f915709398

# 添加新节点
新增一个节点，将节点加入到vsan集群中，新节点可没有存储，只使用它的cpu和内存，也可将磁盘加入vsan集群。
在数据中心处添加节点，并进入维护模式
![](https://qiufuqi.github.io/img/hexo/20221207162952.png)

添加和管理主机，将新主机纳入分布式交换机管理（后期纳入vsan，内部通信）
![](https://qiufuqi.github.io/img/hexo/20221207163141.png)
添加VMkernel适配器，分配内部通信ip
![](https://qiufuqi.github.io/img/hexo/20221207163351.png)
![](https://qiufuqi.github.io/img/hexo/20221207163436.png)
查看新节点 虚拟交换机配置情况
![](https://qiufuqi.github.io/img/hexo/20221207163538.png)

将新节点移入到vsan集群中，资源池根据需要来改变，如果加入失败，参考下一步
![](https://qiufuqi.github.io/img/hexo/20221207163650.png)

声明一块磁盘为闪存盘
选中主机 => 配置 => 存储 => 存储适配器 ，选中设备，标记为闪存盘
![](https://qiufuqi.github.io/img/hexo/20221212134838.png)

新节点退出维护模式，将磁盘加入vsan集群管理
选中集群 => 配置 => vSAN => 磁盘管理 => 声明未使用的磁盘，将新节点的磁盘加入即可
![](https://qiufuqi.github.io/img/hexo/20221207164316.png)

开启vmotion
选中主机 => 配置 => 网络 => VMkernel适配器 => 选中网卡 => 编辑，开启vMotion
![](https://qiufuqi.github.io/img/hexo/20221212135102.png)
![](https://qiufuqi.github.io/img/hexo/20221212135116.png)

## 新节点加入失败
vSphere Web Client 添加主机进VSAN集群时“SAN 主机移至目标群集: vSAN 群集的 UUID 不匹配”报错
原因分析：是因为该esxi主机已经加入过其它集群，和现在新加入的集群UUID冲突了，需要该esxi主机先退出旧集群，然后再加入新集群。
解决方案：
登录该esxi主机的ssh服务器；通过命令行操作。
1、查看VSAN集群状态
执行命令：esxcli vsan cluster get
2、退出VSAN集群
执行命令： esxcli vsan cluster leave
3、加入VSAN集群  -- 可在页面操作
执行命令： esxcli vsan cluster join -u 523ae663-623b-e2fc-39e3-43b15c5ca801

# 删除某节点
选中节点，进入维护模式，迁移全部数据
![](https://qiufuqi.github.io/img/hexo/20221207164612.png)

## 删除网卡
选中节点 => 配置 => 网络 => 虚拟交换机 => 删除VMkernel
![](https://qiufuqi.github.io/img/hexo/20230223143520.png)
![](https://qiufuqi.github.io/img/hexo/20221207164939.png)
分布式交换机删除节点，选择移除节点
![](https://qiufuqi.github.io/img/hexo/20221207165040.png)

删除磁盘组，选中集群 => 配置 => vSAN => 磁盘管理 => 选中节点磁盘，选择移除
![](https://qiufuqi.github.io/img/hexo/20221207165439.png)
选中节点，删除即可
![](https://qiufuqi.github.io/img/hexo/20221207165138.png)

**通过SSH登录节点执行如下操作：**
# 退出集群
可在节点上执行如下操作，方便节点加入别的vsan
1、查看VSAN集群状态
执行命令：esxcli vsan cluster get
2、退出VSAN集群
执行命令： esxcli vsan cluster leave

# 磁盘初始化
[技术参考](https://blog.51cto.com/wangchunhai/5024934)
在vSAN群集中，如果磁盘组中的容量磁盘或缓存磁盘出错，正常情况下可以在vSphere Client的“磁盘管理”界面中移除，如果无法移除时，可以使用ESXi CLI命令行移除。

1、使用Xshell登录到节点ESXi主机，先执行：
esxcli vsan storage list  命令查看缓存磁盘ID和容量磁盘ID（Is SSD: true , VSAN UUID:52d6bb30-6b39-df6f-3543-a4f915709398）
要删除故障或者取消声明的硬盘UUID：52d6bb30-6b39-df6f-3543-a4f915709398
![](https://qiufuqi.github.io/img/hexo/20221208095245.png)
2、执行以下命令移除损坏磁盘
esxcli vsan storage remove -u 52d6bb30-6b39-df6f-3543-a4f915709398
![](https://qiufuqi.github.io/img/hexo/20221208095725.png)