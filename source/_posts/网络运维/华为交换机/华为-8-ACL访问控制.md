---
title: 华为-8-ACL访问控制
date: 2024-2-13
tags:
  - 华为
  - ACL
categories: 
- 运维
- 华为
- ACL
keywords: '华为,ACL'
cover: https://qiufuqi.github.io/img/hexo/20230327084004.png
abbrlink: huawei_acl
url: huawei_acl
comments: false
---
ACL是通过对网络的访问进行控制，通过一些参数匹配特定的流量，并且指明需要对流量进行的操作。
# ACL组成
ACl由一组具有特定顺序的语句组成，设备在查询ACL时，会根据编号从小到大按顺序查找匹配项，一旦发现匹配项就会立即执行与该匹配项关联的动作，停止查找并退出ACL匹配逻辑，每条语句由如下信息构成
- 匹配项：需要匹配的流量。
- 动作：允许或拒绝。
- 编号：决定该语句在ACL中的位置。编号范围：0-4294967294。
  
**每个ACL末尾还有一条隐含语句，这条隐含语句可以匹配任意的数据包**。
# ACL分类和标识
ACL可以从两个角度进行分类：规则定义方式和标识方法。
## 规则定义方式
华为设备会根据网络管理员设置的ACL编号判断ACL的类型
- 基本ACl：2000-2999，基于数据包的源IP地址对流量进行匹配
- 高级ACL：3000-3999，基于数据包的源和目的IP地址，IP协议类型，ICMP类型，TCP/UDP源和目的端口号等第三层和第四层信息对流量进行匹配
- 二层ACL：4000-4999，基于数据帧的源和目的MAC地址，二层协议类型等第二层信息对流量进行匹配
- 用户自定义ACL：5000-5999，基于报文头，便宜位置，字符串掩码等对流量进行匹配

## 标识方法
- 数字型ACL：
- 命名型ACL：管理员可以在命名的同时指定该ACl的编号，设备默认该ACL为高级ACL，从3999开始自动分配一个空闲编号

# ACL匹配机制
按照规则的编号顺序从小到大进行匹配，一旦发现匹配项，立即按照该规则定义的动作进行操作，并终止匹配，退出ACl匹配逻辑。**ACL匹配原则不是最长匹配，而是顺序匹配**。**将更为精确的匹配规则放到较为模糊的匹配规则前。**
- 规则编号按照从小到大的顺序进行匹配
- 一旦匹配成功，立即执行该条规则所定义的动作，终止匹配并退出ACL匹配逻辑
- 编写ACL规则时，需要将更精确的匹配规则放在模糊规则前面
``` bash
# 例如规则15永远不会获得匹配，所以实际应用中应该将更为精确的匹配规则放在较为模糊的匹配规则之前
rule 10 deny source 10.0.0.0 0.0.0.255  # 规则10：拒绝源IP地址----10.0.0.0/24
rule 15 permit source 10.0.0.1 0.0.0.0  # 规则15：放行源IP地址----10.0.0.1/24
rule 30 permit source any               # 规则30：放行源IP地址----任意
```
ACL匹配机制有3中匹配结果：
- 拒绝：deny
- 允许：permit
- 未匹配：路由器会如常处理所有数据包
  - 引用的ACL不存在：在路由器上引用ACL，却发现未在路由器配置中创建ACL
  - ACL中不存在规则：在路由器上引用ACL，但ACL中却没有任何自定义规则
  - ACL中无匹配规则：在路由器上引用ACL，却在ACL中没有匹配上任何规则

# ACL方向
一个路由器的每个接口可以同时设置一个入方向ACL，一个出方向ACL，但一个方向上只能设置一个ACL。

# 基本ACL配置
**基本ACl需要根据源IP地址信息进行匹配**，编号范围2000-2999。**尽可能靠近目的地的位置应用基本ACL**
## 创建基本ACL：
  - 数字型基本ACL：在系统视图中输入命令 acl 2000，即在系统中成功创建了ACL2000，同时进入ACL2000视图
      **acl {number} acl-number [match-order {auto|config}]**
    - number：可选关键词，创建数字型基本ACL时，可省略次关键词，直接输入ACL编号
    - acl-number：ACL的编号，编号范围2000-2999
    - match-order {auto|config}：指定ACL中规则的匹配顺序，**默认是config**。可选参数
      - auto：自动排序，根据深度优先匹配原则，将ACL中配置的多个规则按照精确度从高到底进行排序，并以此顺序对数据包进行匹配。深度优先匹配原则如下：
        - 比较源IP地址范围，源IP地址范围小的规则优先。IP地址通配符掩码中0的位数多的，IP地址范围小
        - 源IP地址相同条件下，规则编号小的优先
      - config：配置顺序，按照管理员指定的规则编号从小到大进行排序
  - 命名型基本ACL：
      **acl name acl-name {basic | acl-number } [match-order {auto|config}]**
    - name：必选关键词，创建命名型ACL是必须使用name关键词
    - acl-name：必选参数，设置命名型基本ACL的名称
    - basic|acl-number：必选参数，任选其一进行配置
      - basic：按照从大到小的顺序选择可用的基本ACL编号进行自动分配，即从2999开始
      - acl-number：系统会根据编号所属的范围来判断ACL的类型
    - match-order {auto|config}：与数字型基本ACL对应字段相同
  
## 配置基本ACL规则：
  基本ACL中可以配置的参数为：源IP地址，是否分段，执行过滤时间
  **rule [rule-id] {deny|permit} [source {source-address source-vildcard | any} | time-range time-name]**
  - rule [rule-id]：设置规则编号，规则编号是一个可选参数，缺省情况下根据默认步长自动进行编号设置，**默认步长是5**。选取第1个编号为步长值5
  - deny|permit：设置规则动作，deny表示拒绝，permit表示允许
  - source {source-address source-vildcard | any}：设置匹配项。必选项。使用源IP地址+通配符掩码方式指定一台主机或者多台主机，any指定任意主机相当于IP地址：0.0.0.0 通配符掩码：255.255.255.255。通配符掩码0表示匹配，1表示不匹配，所以255.255.255.255表示所有位都无须匹配，任意都行。
  - time-range time-name：可选参数，设置ACL规则生效的时间段。time-name是系统中配置的时间段名称。

## 应用基本ACL
  在流量过滤功能中应用基本ACL需要进入接口视图。
  **traffic-filter {inbound|outbound} acl {bas-acl| adv-acl| name acl-name}**
  - inbound|outbound：指明基本ACL的应用方向，inbound表示这是一个入方向的基本ACL，过滤从接口接口收到的流量；outbound表示这是一个出方向的基本ACL，过滤从接口转发的流量
  - acl {bas-acl| adv-acl| name acl-name}：将具体的ACL应用在接口上

``` bash
# 自动设置了步长
[Huawei]acl 2000
[Huawei-acl-basic-2000]rule deny source 192.168.1.0 0.0.0.255
[Huawei-acl-basic-2000]rule permit source any
[Huawei-acl-basic-2000]q
[Huawei]int g0/0/1
[Huawei-GigabitEthernet0/0/1]traffic-filter outbound acl 2000

[Huawei]dis acl 2000
Basic ACL 2000, 2 rules
Acl's step is 5
 rule 5 deny source 192.168.1.0 0.0.0.255 
 rule 10 permit 

```
# 高级ACL配置
基本ACL只能根据数据包的源IP地址进行过滤，存在局限性，所以需要用到高级ACL配置。编号范围3000-3999
## 创建高级ACL
创建高级ACL可分为创建数字型的高级ACL和命名型的高级ACL
- 数字型高级ACL：在系统视图中输入命令 acl 3000，即在系统中成功创建了ACL3000，同时进入ACL3000视图
  **acl [number] acl-number [match-order {auto | config}]**
  - number：可选关键词，创建数字型高级ACL时，可省略此关键词
  - acl-number：ACL的编号，高级ACL的编号范围3000-3999
  - match-order {auto | config}：指定ACL中的匹配规则，默认匹配顺序是config
    - auto：自动排序。根据深度优先的匹配原则，按照精确度从高到底进行排序。深度优先的匹配原则如下：
      - 比较协议范围，指定IP承载的协议规则优先
      - 协议范围相同，比较源IP地址范围，源IP地址范围小的规则优先，源IP地址通配符掩码中0的位数多的，IP地址范围小（**0表示该位需要匹配，1表示不需要匹配**）
      - 协议范围相同，源IP地址范围相同，比较目的IP地址范围，目的IP地址范围小的优先，目的IP地址通配符掩码中0的位数多的，IP地址范围小（**0表示该位需要匹配，1表示不需要匹配**）
      - 协议范围相同，源IP地址范围相同，目的IP地址范围相同，比较第4层端口号（TCP/UDP）范围，端口号范围晓得规则优先
      - 上述条件都相同的话，规则编号小的优先
    - config：按照管理员指定的规则编号从小到大进行排序，没有顺序按照步长。
- 命名型高级ACL：
  **acl name acl-name { advance |acl-number} [match-order {auto|config}]**
  - name：关键词，创建命名型高级ACL时必须使用name关键词
  - acl-name：必选参数，命名型高级ACL名称
  - advance |acl-number：必选参数，任选其一进行配置
    - advance：系统按照从大到小的顺序选择可用的高级ACL编号进行自动分配，即从3999开始分配
    - acl-number：系统会根据编号所属范围来判断ACL的类型
  - match-order {auto|config}：与数字型的高级ACL对应字段相同

## 配置高级ACL规则
### IP协议规则
根据IP承载的协议类型，配置高级ACL规则
**rule [rule-id] {deny|permit} ip [destination {des-address des-wildcard | any} | source {sou-address sou-wildcard | any} |time-range time-name]**
- rule [rule-id]：设置规则编号，可选参数
- deny|permit：规则动作
- ip：设置匹配项为任意IP协议
- destination {des-address des-wildcard | any}：可选项，根据目的IP地址进行匹配，any相当于任意主机
- source {sou-address sou-wildcard | any}：可选项，根据源IP地址进行匹配，any相当于任意主机
- time-range time-name：设置规则生效时间
```bash
# 允许源10.10.10.0/24的流量方位192.168.10.0/24
rule 5 permit destination 192.168.10.0 0.0.0.255 source 10.10.10.0 0.0.0.255
```

### TCP协议类型
协议类型为TCP，配置高级ACL规则
**rule [rule-id] {deny|permit} {protocol-number|tcp} [destination {des-address des-wildcard| any} | destination-port {eq port|gt port|lt port|range port-start port-end} | source {sou-address sou-wildcard|any} source-port {eq port|gt port|lt port|range port-start port-end} |tcp-flag {ack|fin|sys} | time-range time-name]**
- rule [rule-id]：设置规则编号，可选参数
- deny|permit：规则动作
- protocol-number|tcp：指定协议类型为TCP，可以使用**协议号6**来指定，也可以使用关键词tcp指定
- destination {des-address des-wildcard| any}：可选项，根据目的IP地址进行匹配，any相当于任意主机
- destination-port {eq port|gt port|lt port|range port-start port-end}：指定需要匹配的TCP目的端口号或端口号范围，如果不指定表示匹配任意目的端口，eq等于，gt大于，lt小于，range端口号范围
- source-port {eq port|gt port|lt port|range port-start port-end}：指定需要匹配的TCP源端口号或端口号范围，如果不指定表示匹配任意源端口，eq等于，gt大于，lt小于，range端口号范围
- tcp-flag {ack|fin|sys}：指定TCP报文头部中的标记
- time-range time-name：设置规则生效时间

### UDP协议类型
协议类型为UDP时，配置高级ACL规则
**rule [rule-id] {deny|permit} {protocol-number|udp} [destination {des-address des-wildcard| any} | destination-port {eq port|gt port|lt port|range port-start port-end} | source {sou-address sou-wildcard|any} source-port {eq port|gt port|lt port|range port-start port-end} | time-range time-name]**
- rule [rule-id]：设置规则编号，可选参数
- deny|permit：规则动作
- protocol-number|tcp：指定协议类型为TCP，可以使用**协议号17**来指定，也可以使用关键词udp指定
- destination {des-address des-wildcard| any}：可选项，根据目的IP地址进行匹配，any相当于任意主机
- destination-port {eq port|gt port|lt port|range port-start port-end}：指定需要匹配的TCP目的端口号或端口号范围，如果不指定表示匹配任意目的端口，eq等于，gt大于，lt小于，range端口号范围
- source-port {eq port|gt port|lt port|range port-start port-end}：指定需要匹配的TCP源端口号或端口号范围，如果不指定表示匹配任意源端口，eq等于，gt大于，lt小于，range端口号范围
- time-range time-name：设置规则生效时间

## 应用高级ACL
在流量过滤功能中应用高级ACL需要进入到接口视图
**traffic-filter {inbound|outbound} acl {bas-acl|adv-acl|name acl-name}**

``` bash
[Huawei]acl 3000
[Huawei-acl-adv-3000]rule permit ip source 192.168.10.0 0.0.0.255 destination 10
.0.10.0 0.0.0.255
[Huawei-acl-adv-3000]rule permit ip source 192.168.20.0 0.0.0.255 destination 10
.0.10.0 0.0.0.255
[Huawei-acl-adv-3000]rule deny ip
[Huawei-acl-adv-3000]quit
[Huawei]int g0/0/0
[Huawei-GigabitEthernet0/0/0]traffic-filter inbound acl 3000

[Huawei]display acl 3000
Advanced ACL 3000, 3 rules
Acl's step is 5
 rule 5 permit ip source 192.168.10.0 0.0.0.255 destination 10.0.10.0 0.0.0.255 
 rule 10 permit ip source 192.168.20.0 0.0.0.255 destination 10.0.10.0 0.0.0.255
 rule 15 deny ip 

```

# 拓扑实验
通过动态路由OSPF协议实现全网互通
![](https://qiufuqi.github.io/img/hexo/20240213155035.png)
- **display ip routing-table**：查看IP路由表
- **display acl all**：查看配置中所有ACL

## 基本ACL部署
要求：使用基本ACL，禁止PC1访问PC3所属网段设备
- 选择基本ACL部署最佳位置，**尽可能靠近目的地位置应用基本AC**L，避免无意间扩大阻塞范围
  以PC1为源，最佳位置为AR2接口g2/0/0的出口方向，以PC3为源，最佳位置为AR1的g0/0/1的出口方向
- 创建并应用基本ACL
  在AR2的g2/0/0出口方向创建基本ACL
``` bash
# AR2的g2/0/0的出口上
[R2]acl 2000
[R2-acl-basic-2000]rule deny source 10.10.10.0 0.0.0.255
[R2-acl-basic-2000]rule permit source any 
[R2-acl-basic-2000]q
[R2]int g2/0/0
[R2-GigabitEthernet2/0/0]traffic-filter outbound acl 2000

[R2]dis acl all
 Total quantity of nonempty ACL number is 1 
Basic ACL 2000, 2 rules
Acl's step is 5
 rule 5 deny source 10.10.10.0 0.0.0.255 (3 matches)
 rule 10 permit (6 matches)
```

## 高级ACl部署
要求：使用高级ACL，禁止PC2访问PC3所属网段设备
- 选择高级ACL部署最佳位置：高级ACL可以同时匹配源和目的IP地址，无论在哪个接口上应用ACL，都不会扩大阻塞范围。但是为了节省网络带宽资源和网络设备处理资源，在应用高级ACl时，遵循**尽可能靠近源的位置应用高级ACL**
  应用高级ACL的最佳位置是AR1的接口g2/0/0的入口方向
- 创建并应用高级ACL
  在AR1的g2/0/0入口方向创建高级ACL：以PC2为源，PC3为目的
``` bash
# AR1的g2/0/0的入口上
[R1]acl 3000
[R1-acl-adv-3000]rule deny ip destination 10.10.30.0 0.0.0.255 source 10.10.20.0  0.0.0.255
[R1-acl-adv-3000]quit
[R1]int g2/0/0
[R1-GigabitEthernet2/0/0]traffic-filter inbound acl 3000

[R1]display acl 3000
Advanced ACL 3000, 1 rule
Acl's step is 5
 rule 5 deny ip source 10.10.20.0 0.0.0.255 destination 10.10.30.0 0.0.0.255 (2 
matches)
```

## ACL规则顺序
ACL默认是按照顺序来处理每个规则的，一旦发现与数据包相匹配的规则，设备就会执行规则中指定的行为并退出ACL匹配进程。
要求：通过高级ACL禁止AR1的g0/0/0接口ping通AR3的环回接口，但是AR1能够Telnet到AR3的环回接口，同时禁止其他设备Telnet到AR3的环回接口。
- 设计ACL：涉及到两种协议：ICMP和Telnet
  - ICMP：将协议定义为ICMP（或协议号1）
    源IP地址为：10.10.12.1（AR1的g0/0/0接口ip），目的IP地址为：3.3.3.3，AR1发来的ICMP数据包的类型是ICMP请求（Echo-request），因此可以将ICMP指定为echo（echo代表ICMP请求）通配符掩码0.0.0.0可简写为0
    **rule deny icmp source 10.10.12.1 0 destination 3.3.3.3 0 icmp-type echo**
  - Telnet：将协议定义为TCP
    AR1能够Telnet到AR3的环回接口，同时禁止其他设备Telnet到AR3的环回接口，因此需要两条命令。Telnet使用TCP（端口23）
    **rule permit tcp source 10.10.12.1 0 destination 3.3.3.3 0 destination-port eq 23**
    **rule deny tcp source any destination 3.3.3.3 0 destination-port eq 23**
- 创建应用ACL：display查看高级ACL时，发现23已经变为telnet，所以配置时也可以使用关键字telnet，规则15中source any 表示不对源做限制，因此设备将命令写入配置前，通过对命令的解析，省略了这部分
``` bash
[R3]acl 3000
[R3-acl-adv-3000]rule deny icmp source 10.10.12.1 0 destination 3.3.3.3 0 icmp-type echo
[R3-acl-adv-3000]rule permit tcp source 10.10.12.1 0 destination 3.3.3.3 0 destination-port eq 23
[R3-acl-adv-3000]rule deny tcp source any destination 3.3.3.3 0 destination-port eq 23
[R3]int g0/0/1
[R3-GigabitEthernet0/0/1]traffic-filter inbound acl 3000

[R3]display acl 3000
Advanced ACL 3000, 3 rules
Acl's step is 5
 rule 5 deny icmp source 10.10.12.1 0 destination 3.3.3.3 0 icmp-type echo 
 rule 10 permit tcp source 10.10.12.1 0 destination 3.3.3.3 0 destination-port eq telnet 
 rule 15 deny tcp destination 3.3.3.3 0 destination-port eq telnet
```
- 在AR3上启用Telnet：需要现在AR3上启用telnet功能
``` bash
[R3]user-interface vty 0 4
[R3-ui-vty0-4]authentication-mode password 
Please configure the login password (maximum length 16):huawei
```
- 验证配置结果：AR1和AR2分别向AR3的环回接口ping和telnet测试：AR1无法ping通，但是能够telnet通，AR2能ping通，但是无法telnet通
``` bash
# AR1无法ping通，但是能够telnet通
<R1>ping 3.3.3.3
  PING 3.3.3.3: 56  data bytes, press CTRL_C to break
    Request time out
    Request time out

<R1>telnet 3.3.3.3
  Press CTRL_] to quit telnet mode
  Trying 3.3.3.3 ...
  Connected to 3.3.3.3 ...

Login authentication
Password:
<R3>

# AR2能ping通，但是无法telnet通
<R2>ping 3.3.3.3
  PING 3.3.3.3: 56  data bytes, press CTRL_C to break
    Reply from 3.3.3.3: bytes=56 Sequence=1 ttl=255 time=30 ms
    Reply from 3.3.3.3: bytes=56 Sequence=2 ttl=255 time=20 ms
<R2>telnet 3.3.3.3
  Press CTRL_] to quit telnet mode
  Trying 3.3.3.3 ...
  Error: Can't connect to the remote host
```
- 使用自动排序
  华为设备提供了ACL自动排序功能，设备会根据规则之间的重叠和冲突，自动为规则进行排序。设备默认按照管理员配置进行排序，如果需要让设备自行判断规则的顺序，就要**在创建ACL的时候进行指定**，否则当ACL中存在规则时，通过**match-order auto**更改排序时会产生错误。
  在AR3的acl 3000规则中，规则10和规则15的顺序不能颠倒，否则AR1无法对3.3.3.3发起telnet命令，因为数据包会先和deny规则匹配，规则5的顺序与两个规则无关，可以配置在两个规则之后。
```bash
[R3]dis acl 3000
Advanced ACL 3000, 3 rules
Acls step is 5
 rule 5 deny icmp source 10.10.12.1 0 destination 3.3.3.3 0 icmp-type echo (5 matches)
 rule 10 permit tcp source 10.10.12.1 0 destination 3.3.3.3 0 destination-port eq telnet (24 matches)
 rule 15 deny tcp destination 3.3.3.3 0 destination-port eq telnet (2 matches)

[R3]acl 3000 match-order auto
Error: Cannot execute match order command because this ACL is not empty.
```
创建规则时指定自动排序，将上面的规则10和规则15颠倒添加
``` bash
[R3]acl 3000 match-order auto
[R3-acl-adv-3000]rule deny icmp source 10.10.12.1 0 destination 3.3.3.3 0 icmp-type echo
[R3-acl-adv-3000]display this
[V200R003C00]
#
acl number 3000  match-order auto
 rule 5 deny icmp source 10.10.12.1 0 destination 3.3.3.3 0 icmp-type echo 
#
return
```
先添加规则拒绝所有的telnet，发现它的规则编号为10
```bash
[R3-acl-adv-3000]rule deny tcp destination 3.3.3.3 0 destination-port eq telnet
[R3-acl-adv-3000]display this
[V200R003C00]
#
acl number 3000  match-order auto
 rule 5 deny icmp source 10.10.12.1 0 destination 3.3.3.3 0 icmp-type echo 
 rule 10 deny tcp destination 3.3.3.3 0 destination-port eq telnet 
#
return
```
再添加规则允许AR1进行telnet，此时规则与现有的规则10存在重叠和冲突；当添加了与deny规则存在重叠且匹配的IP地址范围更小的permit规则后，ACL自动将匹配范围更精确的permit规则放在了deny前面。此时规则10变成了permit，而deny的规则变为了15。
``` bash
[R3-acl-adv-3000]rule  permit tcp source 10.10.12.1 0 destination 3.3.3.3 0 destination-port eq telnet 
[R3-acl-adv-3000]dis this
[V200R003C00]
#
acl number 3000  match-order auto
 rule 5 deny icmp source 10.10.12.1 0 destination 3.3.3.3 0 icmp-type echo 
 rule 10 permit tcp source 10.10.12.1 0 destination 3.3.3.3 0 destination-port e
q telnet 
 rule 15 deny tcp destination 3.3.3.3 0 destination-port eq telnet 
#
return

```