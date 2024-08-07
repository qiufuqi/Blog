---
title: 华为-5-生成树STP
date: 2024-2-9
tags:
  - 华为
  - STP
categories: 
- 运维
- 华为
- STP
keywords: '华为,STP'
cover: https://qiufuqi.github.io/img/hexo/20230327084004.png
abbrlink: huawei_rstp
url: huawei_rstp
comments: false
---
**生成树协议主要为了防止环路产生**
交换机环路会导致广播风暴，MAC地址震荡（从不同端口学习到MAC地址，不停更新）等后果。交换机在缺省状态下已启用了STP（Spanning Tree Protocol）、RSTP（Rapid Spanning Tree Protocol 快速生成树协议）、MSTP（Multiple Spanning Tree Protocol多生成树协议），无须担心环路产生，但是生产环境中需要对此进行一定的优化，比如确定根桥的位置等等
# STP工作机制：
首先，局域网中的交换机互相交换BPDU（Bridge Protocol Data Unit 桥协议数据单元）消息，选举一台根交换机(Root)，根交换机的所有端口自动成为DP（Designated Port 指定端口）,非根交换机会自动选择距离**根交换机**最近的端口作为RP（Root Port 根端口），然后每条链路会选举一个距离**根交换机**最近的端口作为DP，最后既不是RP也不是DP的端口会被阻塞，形成一个无环路的数据链路网络。
建立STP树后，只有根交换机周期性的发送配置BPDU，非根交换机在收到配置BPDU后通过指定端口把配置BPDU发送出去，实现配置BPDU从根交换机到非根交换机的传输
## BPDU封装字段：  
| PID | PVI | BPDU类型 | 标识 | 根ID | RPC | 桥ID | 端口ID | 消息寿命 | 最大寿命 | Hello时间 |  转发时延 |
- PID：Protocol ID 协议ID，取值为0
- PVI：Protocol Version ID 协议版本ID，取值为0
- BPDU类型：BPDU类型为两种 0x00开头为配置BPDU 配置BPDU从根交换机到各个交换机的传输；0x80的为TCN BPDU 
非根交换机在检测到拓扑变化后，通过根端口生成，并且向根交换机发送
- 标识：标明BPDU的类型
- 根ID：ROOT ID，标识根交换机的桥ID，一台交换机在启动后会把自己的桥ID设置为根ID，向其他交换机发送封装好的BPDU，在收到其他交换机的BPDU后会比较**根ID**，选取最小的最为网络中的根交换机
- RPC：Root Path Cost 根路径开销，标识交换机端口到根交换机的开销。因此在交换机端口选举中，RPC较小的端口会赢得选举。一个启用STP端口的开销值默认与端口的带宽成反比。也可以**手动设置端口的开销值**，一个端口的RPC等于**BPDU从根交换机到这台设备的路径中，经历的所有入站接口的开销之和**
- 桥ID：Bridge ID 由高16为的桥优先级和低48位的mac地址组成。桥优先级有16为，范围0-65536（2的16次方），可以**手动设置桥优先级**，**值必须是4096的整数倍**，默认值是32768，交换机交换BPDU时，桥ID最小的为根路由。
- 端口ID：Port ID，在特定情况下选举端口。端口由高4位的端口优先级和低12位的端口号组成，可以**手动设置端口优先级**，默认值时128，取值范围0-240，**端口优先级必须是16的整数倍**
- 消息寿命：标识配置BPDU从根交换机发出后经历过几台交换机的转发
- 最大寿命：标识交换机经过多长时间没收到BPDU后认为该链路发生了故障，默认值是20s
- Hello时间：根交换机周期性的发送配置BPDU的时间间隔，默认时间是2s
- 转发时延：交换机在侦听(Listening)到学习(learning)状态下停留时间，默认时间是15s

# STP选举流程
- 选举**根交换机**：首先各个交换机将自己的桥ID设置为根ID封装进BPDU中，并且互相交换BPDU，并将收到的BPDU中的根ID和自己的桥ID做比较，选择最小的最为根交换机
- 选择**根端口**：确定根交换机后，非根交换机选择距离根交换机最近的端口最为根端口RP；每台非根交换机会比较所有端口的RPC，如果两个及以上端口的RPC相同，那么非根交换机会选择**对端**桥ID最低的端口，如果多个对端桥ID相同，那么非根交换机选择对端端口ID最低的端口，如果对端端口ID相同，非根交换机比较自己本地端口的端口ID，选择对地端口。**RPC > 对端桥ID（连了不止一根线） > 对端桥端口ID > 本地端口ID**
- 选择**指定端口**：选举规则如下：首先**根交换机的所有端口为指定端口**，其次按照以下顺序排序
  - RPC最低的端口
  - 如果多个端口的RPC相同，那么桥ID最低的端口
  - 如果多个端口的桥ID相同，端口ID最低的被选举为指定端口
- **预备端口**：一旦根交换机，根端口，指定端口选举完成，其余的端口就会被阻塞，也叫做预备端口
- 
# 端口类型： 
此处数据链路指交换机相互连接的物理链路和交换机端口
- 根端口（RP）：在非根交换机上选举出的一个端口，**距离根交换机最近**的端口
  选举原则：（从上到下依次匹配，直到确定）
    - RPC开销最小原则
    如下图所示，交换机2的两个端口RPC分别是g0/0/1=199,g0/0/3=199+200,所以g0/0/1为RP端口，同理交换机3也是
    ![](https://qiufuqi.github.io/img/hexo/20240210085641.png)
    - 对端桥ID优先级高
    如下图所示，交换机4的两个端口的RPC相同，都是199+200，此时比较对端桥ID的优先级即交换机2和交换机3的优先级，此时假定交换机2的优先级更高，则交换机4和交换机2相连的端口g0/0/2为RP
    ![](https://qiufuqi.github.io/img/hexo/20240210090936.png)
    - 对端端口ID最小
    如下图所示，交换机2的三个端口，g0/0/3最先被排除(RPC开销最大)，g0/0/1和g0/0/5开销相同，对端桥ID相同（都是根桥），此时需要比较对端端口ID的大小来确定RP，交换机1连接交换机2的两个端口g0/0/2小一点，所以对应交换机2的g0/0/1端口设置为根端口RP
    ![](https://qiufuqi.github.io/img/hexo/20240210090406.png)
    - 本地端口
    如果对端端口ID也相同，就比较本地交换机的端口，选取最小的为根端口RP
- 指定端口（DP）：**每个数据链路网段中只能有一个指定端口**(一条数据链路上只能有一个指定端口，另一个端口要么为根端口RP，要么为预备端口AP)，根交换机的所有端口为指定端口DP
  选举原则：（从上到下依次匹配，直到确定）
    - RPC开销最小原则
    - 桥ID（交换机优先级）最高
    如下图所示，交换机2的端口g0/0/3=199+200和交换机3的端口g0/0/2=199+200,RPC开销相同，比较桥优先级，此处假定交换机2的优先级更高，则交换机2的端口g0/0/3为指定端口DP，相对应的交换机3的端口g0/0/2设置为预备端口即阻塞端口RP，防止回路产生
    ![](https://qiufuqi.github.io/img/hexo/20240210085641.png)
    - 端口ID优先级最高
    如下图所示，交换机2的端口g0/0/1和g0/0/5端口的RPC相同，桥ID相同(同一个)，g0/0/5端口优先级低一点，所以g0/0/5设定为RP端口
    ![](https://qiufuqi.github.io/img/hexo/20240210090406.png)
- 预备端口（AP）：非根端口且非指定端口，会侦听网段中传输的BPDU（网桥协议数据单元），但不会转发数据

| 端口角色 | 发送BPDU | 接收BPDU | 发送数据 | 接收数据 |
| :------------- | :------------- | :------------- | :------------- | :------------- |
| 根端口RP | 是 | 是 | 是 | 是 |
| 指定端口DP | 是 | 是 | 是 | 是 |
| 预备端口AP | 否 | 是 | 否 | 否 |

根交换机的角色是可以抢占的，如果一个以太网已经选举出根交换机，此时一个根ID值比根交换机更小的交换机接入到网络，那么就会抢占根交换机的角色。一般如果希望某台交换机一直作为根交换机的角色，就将次交换机的桥优先级设置为0。
# STP接口状态
阻塞状态（Blocking），侦听状态（Listening），学习状态（Learning），转发状态（Forwarding）
- 阻塞状态：当一个端口既不是RP，也不是DP，会被阻塞，避免出现环路；接收处理BPDU，不学习MAC地址，不转发BPDU，也不会转发数据帧
- 侦听状态：端口已经是RP或者DP，是一种过度状态，不学习MAC地址，转发BPDU，不转发数据帧，默认情况下该状态持续15s
- 学习状态：侦听结束后的下一个状态，学习MAC地址，转发BPDU，不转发数据，默认情况下该状态持续15s
- 转发状态：转发状态是一个端口履行其正常交换功能的装填，学习MAC地址，转发BPDU，转发数据帧
| 状态 | 接收处理BPDU | 对外发送BPDU | 学习MAC地址 | 转发数据帧 |
| :------------- | :------------- | :------------- | :------------- | :------------- |
| 阻塞状态 | 是 | 否 | 否 | 否 |
| 侦听状态 | 是 | 是 | 否 | 否 |
| 学习状态 | 是 | 是 | 是 | 否 |
| 转发状态 | 是 | 是 | 是 | 是 |

# STP基本实验
实验拓扑图如下
![](https://qiufuqi.github.io/img/hexo/20240209213130.png)
## 查看端口角色
dis stp brief
```bash
# 查看交换机上STP端口角色
dis stp brief
以下示例可看出交换机3为根交换机（Role都是DESI，指定端口），交换机2的G0/0/1端口为阻塞状态（Role是ALTE，状态不是转发状态）
[SW1]dis stp brief
 MSTID  Port                        Role  STP State     Protection
   0    GigabitEthernet0/0/2        DESI  FORWARDING      NONE
   0    GigabitEthernet0/0/3        ROOT  FORWARDING      NONE
[SW2]dis stp brief
 MSTID  Port                        Role  STP State     Protection
   0    GigabitEthernet0/0/1        ALTE  DISCARDING      NONE
   0    GigabitEthernet0/0/3        ROOT  FORWARDING      NONE
[SW3]dis stp brief
 MSTID  Port                        Role  STP State     Protection
   0    GigabitEthernet0/0/1        DESI  FORWARDING      NONE
   0    GigabitEthernet0/0/2        DESI  FORWARDING      NONE
```
## 设置根交换机
stp root primary
- stp enable ：全局启用STP命令，可以在接口状态下启用或关闭undo stp enable
- stp mode stp：改变stp模式，stp，rstp和mstp，默认为mstp
- stp root primary: 将交换机设置为根交换机，它的效果是将设备的优先级更改为0，默认是32768
- stp root secondary: 将交换机设置为备用根交换机，设备优先级更改为4096
- stp priority 数字：设置交换机的设备优先级，数字需要为4096的整数倍
```bash
# 设置交换机为根交换机 连接的交换机stp模式需要一样
stp root primary
设置SW1为根交换机
[SW1]stp enable
[SW1]stp mode stp
Info: This operation may take a few seconds. Please wait for a moment...done.
[SW1]stp root primary

[SW2]stp mode stp
Info: This operation may take a few seconds. Please wait for a moment...done.

[SW3]stp mode stp
Info: This operation may take a few seconds. Please wait for a moment...done.

```
## 查看端口stp状态
dis stp interface 端口号
``` bash
[SW2]dis stp interface g0/0/3
```
## 更改端口开销
端口模式下：stp cost ***
``` bash
[SW2]int g0/0/3
[SW2-GigabitEthernet0/0/3]stp cost 10000
```
## 端口根保护
端口模式下：stp 
```bash
# 启用端口根保护，Protection会显示
[SW2]int g0/0/3
[SW2-GigabitEthernet0/0/3]stp root-protection
[SW2]dis stp brief
 MSTID  Port                        Role  STP State     Protection
   0    GigabitEthernet0/0/1        ROOT  FORWARDING      NONE
   0    GigabitEthernet0/0/3        ALTE  DISCARDING      ROOT
```
## 端口环路保护
端口模式下：stp loop-protection 
```bash
[SW3]int g0/0/2
[SW3-GigabitEthernet0/0/2]stp loop-protection 
[SW3]dis stp brief
 MSTID  Port                        Role  STP State     Protection
   0    GigabitEthernet0/0/1        ROOT  FORWARDING      NONE
   0    GigabitEthernet0/0/2        DESI  FORWARDING      LOOP
```

## BPDU保护和边缘端口
各种各样的终端设备可能连接到网络，这些设备不会发送BPDU以及参与STP运行，通过BPDU保护和设置边缘端口。
- stp bpdu-protection：系统视图下，在交换机上启用BPDU保护
- stp edged-port enable：接口配置，可将端口配置为边缘端口

``` bash
[SW3]stp bpdu-protection
[SW3]int g0/0/1
[SW3-GigabitEthernet0/0/1]stp edged-port enable
```
启用了BPDU保护的边缘端口在收到BPDU后（不小心连接）进入到ERROR DOWN状态，设置30秒后自动恢复
``` bash
[SW3]error-down auto-recovery cause bpdu-protection interval 30
```
# MSTP基本实验
MSTP可以根据vlan来构建无环路路径树，MSTP不仅可以提供备份路径，还可以实现负载分担。
实验拓扑图如下
![](https://qiufuqi.github.io/img/hexo/20240210102156.png)
首先实现基本配置
``` bash
# SW1，SW2，SW3同理，链接端口都设置为trunk，放行所有vlan
[SW1]vlan batch 2 3
Info: This operation may take a few seconds. Please wait for a moment...done.
[SW1]int g0/0/2
[SW1-GigabitEthernet0/0/2]port link-type trunk
[SW1-GigabitEthernet0/0/2]port trunk allow-pass vlan all
[SW1]int g0/0/3
[SW1-GigabitEthernet0/0/3]port link-type trunk
[SW1-GigabitEthernet0/0/3]port trunk allow-pass vlan all
```
## 更改STP运行模式
华为交换机默认的STP运行模式就是MSTP，以下两条命令可以更改为MSTP
- stp mode mstp：将STP模式更改为MSTP
- undo stp mode：将STP模式恢复为默认的MSTP

## MSTP域配置
MSTP加入了域的概念，要想使多个交换机属于同一个MST域，则需要有一些相同的配置
- MST域的域名：默认情况下，MST域名是设备管理口的MAC地址
- MST实例与VLAN的映射关系：默认情况下，MST域内的所有VLAN都映射到MST0
- MST域的修订级别：默认情况下，MST域的修订级别为0，可选配置，若MST域中的所有设备修订级别都是0，则无须修改

配置MST域的步骤如下
- 使用系统视图命令 **stp region-configuration** 进入到MST域视图
- 使用MST域视图命令 **region-name name** 设置MST域的域名，所有设备保持一致
- 使用MST域视图命令 **instance (instnce-id) vlan (vlan id [to vlan id2])** 配置MST实例与VLAN的映射关系，所有设备保持一致
- (可选)使用MST域视图命令 **revision-level (level)** 配置MST域的修订级别，所有设备保持一致
- 使用MST域视图命令 **active region-configuration** 激活MST域的配置，使上述配置全部生效

在三台交换机上分别执行MSTP配置，MSTP域名为HCIA，实例2与vlan2映射，实例3与vlan3映射，更改修订级别为1
配置完成后可以通过 
- **display stp brief**：查看端口状态
- **display stp region-configuration**：查看MSTP参数
- **display stp instance instance-id brief**：查看具体实例的STP状态
- **display stp vlan vlan-id**：查看某个vlan的STP状态
- **display stp interface GigabitEthernet 0/0/1**：查看某个端口的STP状态

```bash
[SW1]stp region-configuration 
[SW1-mst-region]region-name HCIA
[SW1-mst-region]instance 2 vlan 2
[SW1-mst-region]instance 3 vlan 3
[SW1-mst-region]revision-level 1
[SW1-mst-region]active region-configuration 
[SW1]dis stp region-configuration 
 Oper configuration
   Format selector    :0             
   Region name        :HCIA             
   Revision level     :1

   Instance   VLANs Mapped
      0       1, 4 to 4094
      2       2
      3       3
```
通过**dis stp brief**确定，在所有实例中，根交换机都是SW2
``` bash
[SW1]dis stp brief
 MSTID  Port                        Role  STP State     Protection
   0    GigabitEthernet0/0/2        ROOT  FORWARDING      NONE
   0    GigabitEthernet0/0/3        DESI  FORWARDING      NONE
   2    GigabitEthernet0/0/2        ROOT  FORWARDING      NONE
   2    GigabitEthernet0/0/3        DESI  FORWARDING      NONE
   3    GigabitEthernet0/0/2        ROOT  FORWARDING      NONE
   3    GigabitEthernet0/0/3        DESI  FORWARDING      NONE
[SW2-mst-region]dis stp brief
 MSTID  Port                        Role  STP State     Protection
   0    GigabitEthernet0/0/1        DESI  FORWARDING      NONE
   0    GigabitEthernet0/0/3        DESI  FORWARDING      NONE
   2    GigabitEthernet0/0/1        DESI  FORWARDING      NONE
   2    GigabitEthernet0/0/3        DESI  FORWARDING      NONE
   3    GigabitEthernet0/0/1        DESI  FORWARDING      NONE
   3    GigabitEthernet0/0/3        DESI  FORWARDING      NONE
[SW3]dis stp brief
 MSTID  Port                        Role  STP State     Protection
   0    GigabitEthernet0/0/1        ALTE  DISCARDING      NONE
   0    GigabitEthernet0/0/2        ROOT  FORWARDING      NONE
   2    GigabitEthernet0/0/1        ALTE  DISCARDING      NONE
   2    GigabitEthernet0/0/2        ROOT  FORWARDING      NONE
   3    GigabitEthernet0/0/1        ALTE  DISCARDING      NONE
   3    GigabitEthernet0/0/2        ROOT  FORWARDING      NONE
```
## 手动配置根交换机
手动配置根交换机，备用根交换机。
- **stp [instance instancd-id] root primary**：系统视图命令，将当前设备设置为根交换机，如果不指定instance，则会将设备配置为实例0的根交换机
- **stp [instance instance-id] root secondary**：系统视图命令，将当前设备设置为备用根交换机，如果不指定instance，则会将设备配置为实例0的备用根交换机

比如将SW1设置为总根交换机，SW2设置为实例2的根交换机，SW3设置为实例3的根交换机
```bash
[SW1]stp root primary
[SW2]stp instance 2 root primary
[SW3]stp instance 3 root primary 

[SW1]dis stp brief
 MSTID  Port                        Role  STP State     Protection
   0    GigabitEthernet0/0/2        DESI  FORWARDING      NONE
   0    GigabitEthernet0/0/3        DESI  FORWARDING      NONE
   2    GigabitEthernet0/0/2        ROOT  FORWARDING      NONE
   2    GigabitEthernet0/0/3        DESI  FORWARDING      NONE
   3    GigabitEthernet0/0/2        ALTE  DISCARDING      NONE
   3    GigabitEthernet0/0/3        ROOT  FORWARDING      NONE
[SW2]dis stp brief
 MSTID  Port                        Role  STP State     Protection
   0    GigabitEthernet0/0/1        ROOT  FORWARDING      NONE
   0    GigabitEthernet0/0/3        DESI  FORWARDING      NONE
   2    GigabitEthernet0/0/1        DESI  FORWARDING      NONE
   2    GigabitEthernet0/0/3        DESI  FORWARDING      NONE
   3    GigabitEthernet0/0/1        DESI  FORWARDING      NONE
   3    GigabitEthernet0/0/3        ROOT  FORWARDING      NONE
[SW3]dis stp brief
 MSTID  Port                        Role  STP State     Protection
   0    GigabitEthernet0/0/1        ROOT  FORWARDING      NONE
   0    GigabitEthernet0/0/2        ALTE  DISCARDING      NONE
   2    GigabitEthernet0/0/1        ALTE  DISCARDING      NONE
   2    GigabitEthernet0/0/2        ROOT  FORWARDING      NONE
   3    GigabitEthernet0/0/1        DESI  FORWARDING      NONE
   3    GigabitEthernet0/0/2        DESI  FORWARDING      NONE
```
总根交换机各个端口：
![](https://qiufuqi.github.io/img/hexo/20240210102415.png)
实例2根交换机各个端口：
![](https://qiufuqi.github.io/img/hexo/20240210102604.png)
实例3根交换机各个端口：
![](https://qiufuqi.github.io/img/hexo/20240210102724.png)