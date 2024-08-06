---
title: 华为-9-AAA用户管理
date: 2024-2-14
tags:
  - 华为
  - ACL
categories: 
- 运维
- 华为
- AAA
keywords: '华为,AAA,用户管理'
cover: https://qiufuqi.github.io/img/hexo/20230327084004.png
abbrlink: huawei_aaa
url: huawei_aaa
comments: false
---
AAA提供了用户管理功能，是认证（Authentication），授权（Authorization）和计费（Accouting）。认证确保用户身份，授权确保用户所能够执行行为，计费记录用执行的行为，如用户的上线下线时间。
网络设备可以提供本地认证和授权，较大规模网络中，可以设置单独的AAA服务器来提供集中式安全管理，常用的协议是RADIUS（Remote Authentication Dial In User Service，远程认证拨号用户服务）
# AAA基本概念
## 认证
认证功能可以在用户视图访问网络时，识别用户的身份，判断用户是否为合法用户。在对用户进行授权时，应该仅授权用户执行所需功能使得最小权限，以防止发生恶意或意外的网络行为。支持以下3种认证方式：
- 不认证：不对用户的身份进行认证。甚少采用
- 本地认证：使用本地认证时，NAS（Network Access Server）充当了AAA服务器角色，在NAS本地存储了执行用户认证所需的用户信息（例如用户名和密码）。优点：响应速度快；缺点：存储的用户信息量收到NAS的硬件限制，不适合大量存储，因此本地认证主要用于用户登录网络设备进行身份认证，比如Telnet，SSH等
- 远端认证：使用单独的AAA服务器进行身份认证。NAS作为AAA的客户端，使用RADIUS与RADIUS服务器进行通信，或者使用HWTACACS（Huawei Terminal Access Control Access-control System 华为终端访问控制器访问控制系统）与HWTACACS服务器进行通信。可以存储大量的用户凭据信息。用户凭据可能时：密码，用户名和密码，数字证书

## 授权
授权是指授权合法用户能够执行的行为，即授予用户权限。支持3种授权方式：
- 不授权：不对用户进行授权。
- 本地授权：使用本地授权时，NAS实际上充当了AAA服务器的角色，在NAS本地存储了执行授权所需的用户权限信息
- 远端授权：使用单独的AAA服务器进行授权。NAS作为AAA的客户端，使用RADIUS与RADIUS服务器进行通信，或者使用HWTACACS与HWTACACS服务器进行通信。
  - RADIUS授权：通过向RADIUS服务器认证的用户进行授权；**RADIUS的授权功能与其认证功能绑定，不能单独使用**
  - HWTACACS授权：可以向所有用户进行授权

**用户的权限可以同时来自AAA服务器和NAS服务器，当两者的权限范围冲突时，以AAA服务器下发的授权信息为准**
## 计费
计费功能记录用户的行为以及该行为相关的参数，比如时间戳，在线时长，流量等。AAA支持2种计费方式：
- 不计费：用户的行为不产生任何活动日志
- 远端计费：通过RADIUS服务器或HWTACACS服务器进行远端计费。

# AAA实现方式
AAA服务器与AAA客户端通过特殊的协议进行通信，最常用的时RADIUS。RADIUS可以提供对用户的认证，授权和计费功能，这项协议定义在RFC 2865和RFC 2866中。RADIUS将认证和授权绑定。**RADIUS使用UDP传输协议，使用UDP端口1812进行认证和授权，使用UDP端口1813进行计费**

# AAA基本配置
华为通信数字设备支持将不同的认证，授权和计费方案组合使用，也可以使其形成备份，例如在AAA服务器无响应时，使用本地认证。
## 配置本地用户
在设备中创建本地用户，并设置密码。还可以配置本地用户的接入方式，用户级别，闲置切断时间，上线时间，最大连接数量等参数，并且支持本地用户自行修改密码。
  用户接入类型如下：
  - 管理类：通过Telnet，SSH，HTTP，API，FTP等协议进行连接的用户，缺省状态下属于default_admin域
  - 普通类：通过802.1x，PPP和Web进行认证的用户，缺省状态下属于default域
  
  配置用户命令如下：
  - 进入AAA视图：系统视图命令
    **aaa**
  - 创建本地用户：为本地用户设置用户名和密码，用户名可以“用户名@域名”指定用户所属的域
    **local-user user-name password cipher password**
  - 指定接入类型：一个用户可以设置多种接入类型，缺省状态下为none，即关闭所有接入类型
    **local-user user-name service-type { {terminal|telnet|ftp|ssh|snmp|http} | {ppp|none}}**
  - 授权本地用户：授权该用户，定义用户级别，**取值范围0-15，值越大级别越高**
    **local-user user-name privilege level level-num**

## 配置AAA方案
创建并配置认证方案和授权方案
  - 创建认证方案并进入方案：AAA视图命令
    **authentication-scheme authentication-scheme-name**
  - 指定认证方式：缺省状态下为本地认证，可以设置多种认证 (多种认证方式并存)，按照认证方式的采用顺序进行配置
    **authentication-mode {hwtacacs|local|radius|none}**
    - 先本地认证，再远端认证：当用户名只存在远端服务器时，认证方式由本地认证转为远端认证；当本地也保存了用户名时，由于密码错误导致身份认证失败时，不会转为远端认证
    - 先远端认证，再本地认证：再远端服务器无响应时，转为本地认证；当用户名只保存在本地数据库，但远端认证失败时，不会转为本地认证。
  - 创建授权方案并进入授权方案
    **authorization-scheme authorization-scheme-name**
  - 指定授权方式：缺省状态下为本地授权，可以指定多种授权
    **authorizaition-mode {hwhacacs|local|none}**

## 应用AAA方案
针对远程登录进行身份验证时，需要在VTY接口下指定认证方案
  - 进入VTY接口：VTY的接口范围时0-4
    **user-interface vty first-ui-number [last-ui-number]**
  - 指定本地认证
    **authentication-mode aaa**

## 验证命令
  - 查看域中使用的认证，授权和收费方案
    **display domain name default_admin**
  - 查看认证方案配置
    **display authentication-scheme authentication-scheme-name**
  - 查看授权方案配置
    **display authorization-scheme authorization-scheme-name**
  - 查看当前在线用户
    **display users**
  - 查看用户登录信息
    **display aaa offline-record all**

# AAA拓扑实验
![](https://qiufuqi.github.io/img/hexo/20240215110346.png)
基础配置如下：
``` bash
[R1]int g0/0/0
[R1-GigabitEthernet0/0/0]ip add 10.10.12.1 24

[R2]int g0/0/0
[R2-GigabitEthernet0/0/0]ip address 10.10.12.2 24
```
## 查看AAA资源
使用命令display aaa configuration查看是否有足够的资源
``` bash
[R1]display aaa configuration 
  Domain Name Delimiter            : @ 
  Domainname parse direction       : Left to right
  Domainname location              : After-delimiter
  Administrator user default domain: default_admin
  Normal user default domain       : default
  Domain                           : total: 32      used: 2    
  Authentication-scheme            : total: 32      used: 1   
  Accounting-scheme                : total: 32      used: 1   
  Authorization-scheme             : total: 32      used: 1   
  Service-scheme                   : total: 256     used: 0   
  Recording-scheme                 : total: 32      used: 0   
  Local-user                       : total: 512     used: 1  
```

## 配置AAA方案
使用本地方案，配置认证方案和授权方案。配置步骤如下：
- 进入AAA视图：系统视图命令下，输入命令：“aaa”
- 配置认证方案：AAA视图下，输入命令：**authentication-scheme authentication-scheme-name** 创建认证方案，并进入认证方案，缺省情况下设备中存在一个名为default的认证方案。
- 指定认证模式：认证方案视图下，输入命令：**authentication-mode local** 将认证模式指定为本地认证（缺省认证）
- 配置授权方案：AAA视图下，输入命令：**authorization-scheme authorization-scheme-name** 创建授权方案并进入授权方案，缺省情况下设备中存在一个名为default的授权方案，无法删除但可以更改
- 指定授权模式：授权方案视图下，输入命令：**authorization-mode local**将授权模式指定为本地授权（缺省授权）

display authentication-scheme [authentication-scheme-name]：查看配置中的AAA认证方案
display authorization-scheme [authorization-scheme-name]：查看配置中的AAA授权方案
``` bash
[R1]aaa
[R1-aaa]authentication-scheme datacom-authentication
Info: Create a new authentication scheme.
[R1-aaa-authen-datacom-authentication]authentication-mode local
[R1-aaa-authen-datacom-authentication]q
[R1-aaa]authori	
[R1-aaa]authorization-scheme data-authorization
Info: Create a new authorization scheme.
[R1-aaa-author-data-authorization]authorization-mode local

[R1]display authentication-scheme 
  -------------------------------------------------------------------
  Authentication-scheme-name          Authentication-method
  -------------------------------------------------------------------
  default                             Local 
  datacom-authentication              Local 
  -------------------------------------------------------------------
  Total of authentication scheme: 2
[R1]display authorization-scheme 
  -------------------------------------------------------------------
  Authorization-scheme-name          Authorization-method
  -------------------------------------------------------------------
  default                             Local 
  data-authorization                  Local 
  -------------------------------------------------------------------
 Total of authortication-scheme: 2 
```

## 配置业务方案
除了AAA方案，还可以设置一些与管理员用户相关的一些参数，比如管理员用户登录后的级别。每条命令都有其各自对应的级别，只有**用户的级别大于等于命令行的优先级，用户才可以执行这条命令**。
命令级别分别为：0、1、2、3级
- 级别0（参观级）：能够使用ping等网络连通性诊断工具，从本设备向其他设备发出访问命令（比如Telnet），不允许对设备配置进行更改，也无法保存配置为你教案
- 级别1（监控级）：在级别0的基础上，允许用户使用部分display命令，不允许用户保存配置文件
- 级别2（配置级）：用户可以使用配置命令，比如路由配置等，也可以保存配置文件
- 级别3（管理级）：能够使用所有配置命令，还可以使用用于业务故障诊断的debugging命令

配置步骤如下：
- 配置业务方案：AAA视图下，输入命令：**service-scheme service-scheme-name**配置业务方案并进入业务方案视图，默认情况下没有业务方案。
- 指定用户级别：业务方案视图下，输入命令：**admin-user privilege level level-num**指定管理员用户登录后的用户级别，level取值范围为：0-15
``` bash
[R1]aaa
[R1-aaa]service-scheme datacom-service
Info: Create a new service scheme.
[R1-aaa-service-datacom-service]admin-user privilege level 3
```

## 创建自定义管理域
基于域的配置时为了提供更精细化和差异化的AAA服务，华为数通设备中默认有两个域：
- default_admin：全局默认管理域，管理员用户会被划分到这个域中，比如通过Telnet，SSH，FTP，HTTP等方式登录到设备本地的用户
- default：全局默认普通域，接入用户会被划分到这个域中，比如PPP用户，NAC用户等

在未配置自定义域时，用户会被匹配到默认域中，并使用默认域中的各种设置。
配置步骤如下：
- 创建域：AAA视图下，输入命令：**domain domain-name** 创建域并进入域视图
- 应用认证方案：域视图下，输入命令：**authentication-scheme authentication-scheme-name** 应用认证方案
- 应用授权方案：域视图下，输入命令：**authorization-scheme authorization-scheme-name** 应用授权方案
- 应用业务方案：域视图下，输入命令：**service-scheme service-scheme-name** 应用业务方案

**display domain**：查看配置中的域
**display domain name domain-name**：查看某个域的详细信息
``` bash
[R1]aaa
[R1-aaa]domain datacom
Info: Success to create a new domain.
[R1-aaa-domain-datacom]authentication-scheme datacom-authentication 
[R1-aaa-domain-datacom]authorization-scheme data-authorization 
[R1-aaa-domain-datacom]service-scheme datacom-service

[R1]display domain 
  -------------------------------------------------------------------------
  index    DomainName 
  -------------------------------------------------------------------------
  0        default                                                         
  1        default_admin                                                   
  2        datacom                                                         
  -------------------------------------------------------------------------
  Total: 3
[R1]display domain name datacom 

  Domain-name                     : datacom                         
  Domain-state                    : Active
  Authentication-scheme-name      : datacom-authentication
  Accounting-scheme-name          : default
  Authorization-scheme-name       : data-authorization
  Service-scheme-name             : datacom-service
  RADIUS-server-template          : -
  HWTACACS-server-template        : -
  User-group                      : -
```

## 创建本地用户
创建本地用户时可以设置多种参数，比如用户级别，密码复杂度，空闲切断时间等。
配置本地用户命令如下：
- 创建账号密码：AAA视图下，使用命令：**local-user user-name password cipher password** 创建用户，如果有域要求，用户名的完整格式为“用户名@域”。
- 指定接入类型：AAA视图下，使用命令：**local-user user-name service-type telnet** 指定用户接入类型为telnet，一个用户可以指定多种接入类型

**display local-user**：查看设备中的本地用户
**display local-user username user-name**：查看本地用户的详细信息
``` bash
[R1]aaa
[R1-aaa]local-user hcia-admin@datacom password cipher huaweipass
Info: Add a new user.
[R1-aaa]local-user hcia-admin@datacom service-type telnet

[R1]display local-user 
  ----------------------------------------------------------------------------
  User-name                      State  AuthMask  AdminLevel  
  ----------------------------------------------------------------------------
  admin                          A      H         -          
  hcia-admin@datacom             A      T         -          
  ----------------------------------------------------------------------------
  Total 2 user(s)
[R1]display local-user username hcia-admin@datacom
  The contents of local user(s):
  Password          : ****************
  State             : active    
  Service-type-mask : T
  Privilege level   : -
  Ftp-directory     : - 
  Access-limit      : -        
  Accessed-num      : 0   
  Idle-timeout      : -
  User-group        : -
```
AuthMask指的是本地用户的接入类型：
- T：Telnet用户
- M：终端用户，通常指Console用户
- S：SSH用户
- F：FTP用户
- W：Web用户
- X：802.1x用户
- A：用户可以使用所有的接入类型
- H：HTTP用户
- 组合类型：比如TH，表示可以使用Telnet和HTTP方式接入

## 配置Telnet功能
启用路由器的Telnet功能，并且将Telnet访问的用户设置成本地AAA认证和授权。
通过VTY登录的用户进行认证，认证有三种方式：
  - AAA认证：用户登录时输入账号密码，设备根据本地配置的AAA方案对用户进行认证
  - 密码认证：用户在登录时输入认证密码，设备根据本地配置的认证密码对用户进行认证
  - 不认证：不需要输入任何认证信息，直接登录设备

配置步骤如下：
- 启用Telnet功能：系统视图命令
  **telnet server enable**
  **display telnet server status**：查看telnet服务器状态
``` bash
[R1]telnet server enable
 Error: TELNET server has been enabled
[R1]display telnet server status
 TELNET IPV4 server                      :Enable
 TELNET IPV6 server                      :Enable
 TELNET server port                      :23
```
- 配置AAA认证：
user-interface vty first-ui-number last-ui-number：进入VTY用户界面视图
authentication-mode aaa：VTY视图命令，指定VTY用户的认证模式为AAA认证
```bash
[R1]user-interface vty 0 4
[R1-ui-vty0-4]authentication-mode aaa
```

## 验证配置
在AR2上使用telnet命令登录AR1
**telnet IP**：通过telnet登录
**display users**：查看登录设备的用户
**display access-user**：查看设备上已存在的用户连接
**display access-user user-id user-id-num**：查看该用户的详细信息，用户名，所属域，IP地址，AAA认证类型和AAA方案
``` bash
<R2>telnet 10.10.12.1
  Press CTRL_] to quit telnet mode
  Trying 10.10.12.1 ...
  Connected to 10.10.12.1 ...

Login authentication


Username:hcia-admin@datacom
Password:
<R1>
<R1>display users
  User-Intf    Delay    Type   Network Address     AuthenStatus    AuthorcmdFlag
  0   CON 0   00:02:09                                   pass                   
  Username : Unspecified

+ 129 VTY 0   00:00:00  TEL    10.10.12.2                pass                   
  Username : hcia-admin@datacom  

<R1>display access-user
 ------------------------------------------------------------------------------
 UserID Username                       IP address                   MAC  
 ------------------------------------------------------------------------------
 1      hcia-admin@datacom             2.12.10.10                -              

 ------------------------------------------------------------------------------
 Total 1,1 printed
<R1>display access-user user-id 1
Basic:
  User id                         : 1
  User name                       : hcia-admin@datacom
  Domain-name                     : datacom                         
  User MAC                        : -
  User IP address                 : 2.12.10.10
  User access time                : 2024/02/15 16:46:35
  User accounting session ID      : R1002552550000000000e4ada000001
  User access type                : Telnet
  Idle Timeout                    : 4294967236(s)

AAA:
  User authentication type        : Administrator authentication
  Current authentication method   : Local
  Current authorization method    : Local
  Current accounting method       : None
```
**free user-interface ui-type ui-number**：断开某个用户，先用display users确认用户的登录方式
**display aaa statistics offline-reason**：查看用户下线的原因统计
``` bash
<R1>dis users
  User-Intf    Delay    Type   Network Address     AuthenStatus    AuthorcmdFlag
+ 0   CON 0   00:00:00                                   pass                   
  Username : Unspecified

  129 VTY 0   00:00:21  TEL    10.10.12.2                pass                   
  Username : hcia-admin@datacom  

<R1>free user-interface vty 0
Warning: User interface vty0 will be freed. Continue? [Y/N]:y
 [OK]
<R1>
Feb 15 2024 16:54:18-08:00 R1 %%01LINE/3/CLR_ONELINE(l)[0]:The user chose Y when
 deciding whether to disconnect the specified user interface. 

<R1>display aaa statistics offline-reason 
  19  user request to offline       :2
```