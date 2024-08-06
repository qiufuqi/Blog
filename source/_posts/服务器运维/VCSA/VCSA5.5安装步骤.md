---
title: VCSA5.5安装步骤.md
date: 2023-3-15
tags:
  - EXSI
  - vCenter
categories: 
- 运维
- EXSI
- vCenter
keywords: 'Exsi,vCenter'
cover: https://qiufuqi.github.io/img/hexo/20221207140120.png
abbrlink: exsi_vCenter_5.5
comments: false
---

**VCSA5.5 安装步骤**

VCSA 用于管理EXSI虚拟化软件（虚拟化版本不能高于VCSA，即 VCSA版本为7.0 则最高只能管理EXSI7）
VCSA5.5 比较特殊，安装步骤如下
# 下载对应文件
名称VMware-vCenter-Server-Appliance-5.5.0.31100-9911210_OVF10.ova，这里是5.5版本，你可以理解为这个是已经被包装过的镜像文件模版。
镜像下载地址分享
链接: https://pan.baidu.com/s/1ONjVivtoc0TYmtK0vIhDKw?pwd=wak9 
提取码: wak9 

其他镜像下载：https://www.dinghui.org/vmware-iso-download.html

安装参考：https://www.renrendoc.com/paper/216758580.html

# 导入模板
新建虚拟机，选择从OVF或OVA文件部署虚拟机
![](https://qiufuqi.github.io/img/hexo/20230315141842.png)
命名系统并上传镜像，等待系统导入
![](https://qiufuqi.github.io/img/hexo/20230315141958.png)
# 开机配置
系统开机，并登录，使用默认用户名**root**，默认密码为**vmware**进行登录
![](https://qiufuqi.github.io/img/hexo/20230315142740.png)

# 配置IP
命令行输入 ./opt/vmware/share/vami/vami_config_net
![](https://qiufuqi.github.io/img/hexo/20230315142903.png)
设置IP，网关等
![](https://qiufuqi.github.io/img/hexo/20230315143618.png)
配置完成后输入exit，按照步骤操作 
![](https://qiufuqi.github.io/img/hexo/20230315150315.png)

# 开始安装
win7 系统 IE浏览器 按照步骤开始安装
![](https://qiufuqi.github.io/img/hexo/20230315150636.png)
同意协议
![](https://qiufuqi.github.io/img/hexo/20230315150823.png)

默认配置，把数据库和SSO软件都部署在本虚拟机，且不安装AD域和时间同步
![](https://qiufuqi.github.io/img/hexo/20230315150920.png)
选择默认，开始安装，显示各个组件安装状态
![](https://qiufuqi.github.io/img/hexo/20230315151139.png)

# 访问系统
安装完成后，修改密码（旧密码vmware）
![](https://qiufuqi.github.io/img/hexo/20230315151649.png)

浏览器输入IP地址，可选择客户端登录或者UI登录（UI登录需要flash，这里建议使用vsphere登录管理）
![](https://qiufuqi.github.io/img/hexo/20230315152034.png)

输入IP，账号密码
![](https://qiufuqi.github.io/img/hexo/20230315153933.png)
![](https://qiufuqi.github.io/img/hexo/20230315153902.png)

输入许可证
![](https://qiufuqi.github.io/img/hexo/20230315153834.png)
``` bash
VCSA许可证：
VMware vCenter 7.0 Standard
104HH-D4343-07879-MV08K-2D2H2
410NA-DW28H-H74K1-ZK882-948L4
406DK-FWHEH-075K8-XAC06-0JH08

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
ESXi 7.0 Enterprise Plus
JJ2WR-25L9P-H71A8-6J20P-C0K3F
HN2X0-0DH5M-M78Q1-780HH-CN214
JH09A-2YL84-M7EC8-FL0K2-3N2J2

EXSI6：
0A65P-00HD0-3Z5M1-M097M-22P7H

EXSI5.5：
UJ623-A8091-K8L4T-0P38H-8ENQ1
4F6GR-A6K9Q-LZPC8-Z1AQ4-AA34U

EXSI5.1：
JA0XT-4W300-AZDW9-43856-8A8HL
```