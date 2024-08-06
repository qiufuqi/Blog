---
title: VCSA安装步骤
date: 2022-09-21
tags:
  - EXSI
  - vCenter
categories: 
- 运维
- EXSI
- vCenter
keywords: 'Exsi,vCenter'
cover: https://qiufuqi.github.io/img/hexo/20220922143047.png
abbrlink: exsi_vCenter_install
comments: false
---

**Vmware vCenter 6.7部署安装全过程**

[参考地址](https://baijiahao.baidu.com/s?id=1735797791474009805&wfr=spider&for=pc)

镜像下载地址分享
链接: https://pan.baidu.com/s/1ONjVivtoc0TYmtK0vIhDKw?pwd=wak9 
提取码: wak9 

其他镜像下载：https://www.dinghui.org/vmware-iso-download.html

# 安装准备
前期准备：
- 安装前请先准备好VMware-VCSA-all-6.7.0-19300125.iso文件，放置在windows操作系统下。
- 一台EXSI主机 （vCenter系统安装在次主机上）
  
vCenter硬件要求
准备把vCenter安装在ESXI服务器上的一个虚拟机中，最低12G内存+2VCPU，这个条件还只是最小化安装vCenter，只能管理10台ESXI服务器，100台虚拟机。由于ESXI本身需要4G内存，因此使用此搭配时，至少需要16G+的内存，否则无法安装虚拟机。

# 安装开始
双击前面准备好的iso文件，将其挂载为DVD驱动器，进入vcsa-ui-installer文件夹。
![](https://qiufuqi.github.io/img/hexo/20220921172408.png)
然后选择win32，如果你的其它的操作系统，则相应安装。
![](https://qiufuqi.github.io/img/hexo/20220921172436.png)
双击安装程序，开始安装。
![](https://qiufuqi.github.io/img/hexo/20220921172455.png)
设置安装界面语言为中文，点击右上角切换，点击“安装”。
![](https://qiufuqi.github.io/img/hexo/20220921172517.png)

下一步
![](https://qiufuqi.github.io/img/hexo/20220921172626.png)
勾选许可，点击下一步
![](https://qiufuqi.github.io/img/hexo/20220921172704.png)

输入ESXi主机的地址、端口号、用户名以及密码，然后点击下一步  提前准备的exsi主机
![](https://qiufuqi.github.io/img/hexo/20220921172728.png)
点击是
![](https://qiufuqi.github.io/img/hexo/20220921172809.png)
设置 一下虚拟名称，root密码，点击下一步。
![](https://qiufuqi.github.io/img/hexo/20220921172829.png)
我这里就部署一台微型，点击下一步。不同部署大小所需的资源如下图：
![](https://qiufuqi.github.io/img/hexo/20220921172846.png)
下一步
![](https://qiufuqi.github.io/img/hexo/20220921172903.png)
根据自己的情况选择dhcp或是静态ip地址，配置。下一步
![](https://qiufuqi.github.io/img/hexo/20220921172920.png)
最后点击完成。
![](https://qiufuqi.github.io/img/hexo/20220921172934.png)
![](https://qiufuqi.github.io/img/hexo/20220921172945.png)
![](https://qiufuqi.github.io/img/hexo/20220921172958.png)
继续部署第二阶段，点击继续。然后点击下一步
![](https://qiufuqi.github.io/img/hexo/20220921173022.png)
点击下一步
![](https://qiufuqi.github.io/img/hexo/20220921173037.png)
输入vCenterServer密码，并单击”下一步”
![](https://qiufuqi.github.io/img/hexo/20220921173051.png)
取消勾选，点击下一步。
![](https://qiufuqi.github.io/img/hexo/20220921173112.png)
点击完成
![](https://qiufuqi.github.io/img/hexo/20220921174019.png)
点击确定
![](https://qiufuqi.github.io/img/hexo/20220921174035.png)
最后耐心等待即可。
![](https://qiufuqi.github.io/img/hexo/20220921174105.png)
完成后点击关闭
![](https://qiufuqi.github.io/img/hexo/20220921174122.png)

然后通过我们的浏览器就能正常访问了。
![](https://qiufuqi.github.io/img/hexo/20220921174214.png)

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

# 新建数据中心
添加主机了。点击首页IP地址后面的“操作”下拉框，选择“新建数据中心”。
![](https://qiufuqi.github.io/img/hexo/20220921174734.png)
设置新建数据中心的名称。
![](https://qiufuqi.github.io/img/hexo/20220921174755.png)

新建完成之后，点击进入到数据中心。再点击数据中心后面的“操作”下拉框，选择“添加主机”。
![](https://qiufuqi.github.io/img/hexo/20220921174813.png)

设置ESXi主机的IP地址信息，按照提示进行下一步
![](https://qiufuqi.github.io/img/hexo/20220921174829.png)
分配许可证信息。
![](https://qiufuqi.github.io/img/hexo/20220921174910.png)

锁定模式，一般不建议启用，如果是正规的数据中心，应该要进行管控。我就这一台，而且vCenter还是嵌入式部署，可不敢点启用，使用了默认的“禁用”。
![](https://qiufuqi.github.io/img/hexo/20220921174931.png)

再点击到数据中心“Guo Tiejun”下的主机，就可以看到主机信息了。
![](https://qiufuqi.github.io/img/hexo/20220921175001.png)

# 其他问题处理
## 管理问题
vcenter在添加主机时，锁定模式选择“正常”，故导致无法直接登陆ESXI宿主机
![](https://qiufuqi.github.io/img/hexo/20220921175051.png)

在vcenter选择相应的ESXI宿主机，配置>系统>安全配置文件>锁定模式>编辑，修改为禁用。
![](https://qiufuqi.github.io/img/hexo/20220921175125.png)

## 迁移问题
[迁移参考](https://blog.51cto.com/xiaoyuanzheng/5611330)
迁移的两种：冷迁移和热迁移。
冷迁移：vm关机后进行迁移，该方法适用所有vm，只要不是vm的版本不高过esxi的所支持版本，迁移后都可以正常启动
热迁移：vm在开机状态下进行迁移。

vmware从6.7版本开始支持跨cpu进行热迁移了，只需要开启一个叫evc的功能。（6.7以前没这个功能）
开启evc功能后，分intel跟amd的cpu支持热迁移。选择之后，vm使用的cpu是vmware虚拟机一层后的cpu，也就是模拟intel或者amd的cpu，这个时候cpuid 还有特性会被修改，不依赖物理主机的cpu。

### 开启evc功能
开启evc的位置有两个，一个是针对单个vm的,需要在关机状态下才能操作，开机状态无法操作，开启路径选择一台vm——配置——vmware evc —— 编辑–指定cpu，如下图：
![](https://qiufuqi.github.io/img/hexo/20221104085507.png)

第二个开启evc的位置是通过群集，选择群集–配置——vmware evc —— 编辑–指定cpu，但是群集启用有要求，必须是群集里面没有开启其他系列evc的vm。 也就是说，开启的时候，会应用到群集所有vm，但是要求vm都没开启evc（如开启evc，必须是选择同一个系列的cpu，例如都选择了intel–sky 系列cpu）。
开启之后，vm重启后会生效evc的功能。

这里注意一点，如果把群集的evc功能关闭，群集内的vm的evc功能并不会失效，即便重启vm也不会关闭，具体原因不知道，所以建议尽量在单个vm上开启evc，最好是做镜像的时候设置，不要开启群集的evc。

**解决DELL 服务器安装完esxi后evc模式被禁用问题**
当我们安装完DELL服务器的esxi后，主机EVC模式出现：CPU模式 不可用（无响应）时，是因为在BIOS里面【Monitor/Mwait】功能被禁用了，需要开启。
![](https://qiufuqi.github.io/img/hexo/20221104085646.png)
开启
![](https://qiufuqi.github.io/img/hexo/20221104085658.png)


### 开启vMotion
单击主机→配置→VMkernel适配器→单击vmk0设备→编辑
![](https://qiufuqi.github.io/img/hexo/20220926103544.png)
勾选vMotion服务，然后确定 
![](https://qiufuqi.github.io/img/hexo/20220926103615.png)


输入许可证
``` bash
VCSA许可证：
VCSA7:
JC45H-0Z292-M8808-1C0Q0-3A8H8

VCSA6:
HG612-FH19H-08DL1-V19X2-1VKND
NU4JA-4V2DQ-48428-T32GK-8VRN4
0Y4H2-8P217-H8900-M8AE4-2LH44
NA658-2308J-08809-93AQ6-278J0

VCSA5：
JG6QQ-02L92-UZR39-R13Q2-120LC


EXSI许可证：
EXSI6：
0A65P-00HD0-3Z5M1-M097M-22P7H

EXSI5.5：
UJ623-A8091-K8L4T-0P38H-8ENQ1
4F6GR-A6K9Q-LZPC8-Z1AQ4-AA34U

EXSI5.1：
JA0XT-4W300-AZDW9-43856-8A8HL
```