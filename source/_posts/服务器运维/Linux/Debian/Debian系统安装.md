---
title: Debian系统安装
date: 2023-06-06
tags:
  - Linux
  - 系统
  - Debian
categories: 
- 运维
- 系统
- Debian
keywords: 'Linux,Debian'
description: Debian系统安装
cover: https://qiufuqi.github.io/img/hexo/20230606165524.png
abbrlink: debian_install
comments: false
---

Debian系统安装
在本机VMware Workstation Pro或者服务器虚拟化环境中创建虚拟机，并挂载镜像，安装建议安装英文版，本次实验使用中文版方便对照

## 启动镜像
启动镜像，选择图形安装界面，回车：
![](https://qiufuqi.github.io/img/hexo/20230606165953.png)
## 选择语言
这里选择 Chineses 中文简体语言，然后点击 Continue：（建议English）
![](https://qiufuqi.github.io/img/hexo/20230606170131.png)
## 选择地区
地区选择中国
![](https://qiufuqi.github.io/img/hexo/20230606170229.png)
## 选择键盘
默认 中文，可以更改American English，然后点击 Continue：
![](https://qiufuqi.github.io/img/hexo/20230606170355.png)
## 配置网络
![](https://qiufuqi.github.io/img/hexo/20230606170541.png)
选择手动网络设置
![](https://qiufuqi.github.io/img/hexo/20230606170622.png)
输入指定的IP/掩码，网关，域名服务器（根据自身情况来）
![](https://qiufuqi.github.io/img/hexo/20230606170715.png)
![](https://qiufuqi.github.io/img/hexo/20230606170731.png)
![](https://qiufuqi.github.io/img/hexo/20230606170801.png)
输入主机名，域名
![](https://qiufuqi.github.io/img/hexo/20230606170849.png)
![](https://qiufuqi.github.io/img/hexo/20230606170922.png)

## 设置root密码
两次密码需要保持一致，设置完 root 密码，点击 Continue：
![](https://qiufuqi.github.io/img/hexo/20230606171007.png)
## 设置普通用户和密码
![](https://qiufuqi.github.io/img/hexo/20230606171051.png)
![](https://qiufuqi.github.io/img/hexo/20230606171118.png)

## 磁盘分区
磁盘分区，可以使用向导或者手动分区，推荐使用第二种方式
![](https://qiufuqi.github.io/img/hexo/20230606171223.png)

### 方式一：使用整个磁盘
选中该选项，回车，选中该磁盘
![](https://qiufuqi.github.io/img/hexo/20230606171401.png)
设置分区，区分不同分区存放位置，本次选择/home放在单独分区
![](https://qiufuqi.github.io/img/hexo/20230606171501.png)

进行分区确认信息，如果确认，选中**结束分区设定并将修改写入磁盘**
![](https://qiufuqi.github.io/img/hexo/20230606171601.png)

### 方式二：使用整个磁盘配置LVM
选中“分区向导”，可重新选择分区方式
选中该选项，回车，选中该磁盘
![](https://qiufuqi.github.io/img/hexo/20230606171401.png)
设置分区，区分不同分区存放位置，本次选择/home放在单独分区
![](https://qiufuqi.github.io/img/hexo/20230606171501.png)
设置LVM需要写入磁盘，并设置LVM最大空间
![](https://qiufuqi.github.io/img/hexo/20230606172229.png)
![](https://qiufuqi.github.io/img/hexo/20230606172416.png)

进行分区确认信息，如果确认，选中**结束分区设定并将修改写入磁盘**
![](https://qiufuqi.github.io/img/hexo/20230606172448.png)

PS：选中某个分区，回车可进行修改：**用途，挂载点（名称/home）等信息，或者删除重新挂载**
![](https://qiufuqi.github.io/img/hexo/20230606172658.png)

### 方式三：手动分区
手动分区可参考以下容量进行，/根目录一般大一点
![](https://qiufuqi.github.io/img/hexo/20230609091434.png)



以上三种方式都可以设置磁盘分区，最终确定分区方案，确认信息
![](https://qiufuqi.github.io/img/hexo/20230606173759.png)

## 安装系统
保存完磁盘分区即开始安装基本操作系统，等待安装完成
![](https://qiufuqi.github.io/img/hexo/20230606174021.png)

## 包管理器
选择 no，不搜索其他安装介质，点击 Continue
![](https://qiufuqi.github.io/img/hexo/20230606174424.png)
选择 no，不使用网络镜像(慢)
![](https://qiufuqi.github.io/img/hexo/20230606174449.png)
选择 no，不参加软件包流行度调查
![](https://qiufuqi.github.io/img/hexo/20230606174722.png)

## 安装软件
可根据需要进行安装
![](https://qiufuqi.github.io/img/hexo/20230606174808.png)

## 安装 grub 引导程序
这里需要选择 Yes，不然进入不了系统
![](https://qiufuqi.github.io/img/hexo/20230606180420.png)
选择 grub 引导程序安装位置
![](https://qiufuqi.github.io/img/hexo/20230606180435.png)
完成系统安装
![](https://qiufuqi.github.io/img/hexo/20230606180539.png)

## 问题处理
### 更换源
debian默认从镜像源更新，由于我们光驱中并没有放置安装光盘，因此需要修改镜像源
apt-get update
替换镜像源，注释掉原有的镜像源，这里存放了阿里云、腾讯云、网易三个镜像源，只启用阿里云的镜像源，如下修改：
``` bash 
root@debian:~# cat /etc/apt/sources.list
# deb cdrom:[Debian GNU/Linux 11.3.0 _Bullseye_ - Official amd64 DVD Binary-1 20220326-11:23]/ bullseye contrib main
 
#deb cdrom:[Debian GNU/Linux 11.3.0 _Bullseye_ - Official amd64 DVD Binary-1 20220326-11:23]/ bullseye contrib main
 
#deb http://security.debian.org/debian-security bullseye-security main contrib
#deb-src http://security.debian.org/debian-security bullseye-security main contrib
 
# bullseye-updates, to get updates before a point release is made;
# see https://www.debian.org/doc/manuals/debian-reference/ch02.en.html#_updates_and_backports
# A network mirror was not selected during install.  The following entries
# are provided as examples, but you should amend them as appropriate
# for your mirror of choice.
#
# deb http://deb.debian.org/debian/ bullseye-updates main contrib
# deb-src http://deb.debian.org/debian/ bullseye-updates main contrib
 
 
#Aliyun Apt Source 阿里云镜像源
deb http://mirrors.aliyun.com/debian/ bullseye main contrib
deb-src http://mirrors.aliyun.com/debian/ bullseye main contrib
deb http://mirrors.aliyun.com/debian/ bullseye-updates main contrib
deb-src http://mirrors.aliyun.com/debian/ bullseye-updates main contrib
deb http://mirrors.aliyun.com/debian/ bullseye-backports main contrib
deb-src http://mirrors.aliyun.com/debian/ bullseye-backports main contrib
deb http://mirrors.aliyun.com/debian/ bullseye-proposed-updates main contrib
deb-src http://mirrors.aliyun.com/debian/ bullseye-proposed-updates main contrib
 
 
#Tencent Apt Source 腾讯云镜像源
#deb https://mirrors.tencent.com/debian/ bullseye main non-free contrib
#deb-src https://mirrors.tencent.com/debian/ bullseye main non-free contrib
#deb https://mirrors.tencent.com/debian-security/ bullseye-security main
#deb-src https://mirrors.tencent.com/debian-security/ bullseye-security main
#deb https://mirrors.tencent.com/debian/ bullseye-updates main non-free contrib
#deb-src https://mirrors.tencent.com/debian/ bullseye-updates main non-free contrib
#deb https://mirrors.tencent.com/debian/ bullseye-backports main non-free contrib
#deb-src https://mirrors.tencent.com/debian/ bullseye-backports main non-free contrib
 
#163 Apt Source 网易镜像源
#deb https://mirrors.163.com/debian/ bullseye main non-free contrib
#deb-src https://mirrors.163.com/debian/ bullseye main non-free contrib
#deb https://mirrors.163.com/debian-security/ bullseye-security main
#deb-src https://mirrors.163.com/debian-security/ bullseye-security main
#deb https://mirrors.163.com/debian/ bullseye-updates main non-free contrib
#deb-src https://mirrors.163.com/debian/ bullseye-updates main non-free contrib
#deb https://mirrors.163.com/debian/ bullseye-backports main non-free contrib
#deb-src https://mirrors.163.com/debian/ bullseye-backports main non-free contrib
```
重新更新镜像源缓存  update是更新软件列表，upgrade是更新软件。
``` bash
# update是更新软件列表，upgrade是更新软件。
root@debian:~# apt-get update
root@debian:~# apt-get upgrade
```
### root无法登录
debian10向上版本创建虚拟机时我们设置了root账户密码，然而在登入时却在未列出中无法登入root账户
1. 普通账号登录后打开终端窗口，输入su - root，切换root身份
2. vi  /etc/pam.d/gdm-password，找到"auth  required  pam_succeed_if.so user !=root quiet_success"，修改登录pam文件，在行最前面加#
保存退出后就可以使用root账号登录了

### SSH设置root允许登录
全新安装系统后，默认情况下将禁用Debian Linux上的root登录。当您尝试以root用户身份登录Debian Jessie Linux服务器时，访问将被拒绝。
要在Debian Linux系统上为root用户启用SSH登录，您需要首先配置SSH服务器。打开/etc/ssh/sshd_config并更改以下行：
``` bash
# 安装ssh
apt-get install -y ssh
```
从：
PermitRootLogin without-password
至：
PermitRootLogin yes
完成上述更改后，请重新启动SSH服务器：
``` bash
# /etc/init.d/ssh restart
[ ok ] Restarting ssh (via systemctl): ssh.service.
```
从现在开始，您将能够以root用户身份使用ssh登录