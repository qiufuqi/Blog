---
title: CentOS系统安装
date: 2023-06-07
tags:
  - Linux
  - 系统
  - CentOS
categories: 
- 运维
- 系统
- CentOS
keywords: 'Linux,CentOS'
description: CentOS系统安装
cover: https://qiufuqi.github.io/img/hexo/20230607144159.png
abbrlink: centos_install
comments: false
---

**CentOS系统安装**
在本机VMware Workstation Pro或者服务器虚拟化环境中创建虚拟机，并挂载镜像，安装建议安装英文版，本次实验使用中文版方便对照

## 启动镜像
启动镜像，选择安装CentOS7
![](https://qiufuqi.github.io/img/hexo/20230607144644.png)

## 选择语言
这里选择 Chineses 中文简体语言，然后点击 Continue：（建议English）
![](https://qiufuqi.github.io/img/hexo/20230607144848.png)

## 安装信息配置
### 本地化
#### 设置日期和时间
点击”日期和时间“，这里选择 亚洲 上海
![](https://qiufuqi.github.io/img/hexo/20230607145703.png)
![](https://qiufuqi.github.io/img/hexo/20230607145906.png)

### 软件
#### 软件安装
软件安装默认是最小安装，可根据自己需要选择要安装的组件
![](https://qiufuqi.github.io/img/hexo/20230607150203.png)

### 系统信息
#### 安装位置
系统安装位置是一定要选择
方式一：自动分区 安装程序提供默认分区，自动分区的根目录一般默认最多50G
自动分区则选中 "自动配置分区" 点击完成即可

方式二：手动分区 当用户根目录大小有要求，或者创建指定的目录名称时
手动分区则选中 "我要配置分区" 点击完成可跳转到手动分区
![](https://qiufuqi.github.io/img/hexo/20230607150813.png)
到达手动分区，可以手动一个一个创建新的挂载点，也可以自动创建(自动创建相当于自动分区，但可以调整大小，推荐)
![](https://qiufuqi.github.io/img/hexo/20230607150853.png)
接受更改即可
![](https://qiufuqi.github.io/img/hexo/20230607151142.png)

#### 设置网络和主机名
先设置网络，IP相关信息可联系网络组。
点击”网络和主机名“
![](https://qiufuqi.github.io/img/hexo/20230607145015.png)
点击”配置“
![](https://qiufuqi.github.io/img/hexo/20230607145108.png)
选择 IPv4设置，方法：手动；Add添加地址并填写具体地址信息
![](https://qiufuqi.github.io/img/hexo/20230607145338.png)

设置好网络并打开网络连接，编辑主机名，点击完成
![](https://qiufuqi.github.io/img/hexo/20230607145507.png)

## 开始安装
设置root密码和创建用户，完整完成后重启服务器
![](https://qiufuqi.github.io/img/hexo/20230607151320.png)
![](https://qiufuqi.github.io/img/hexo/20230607151301.png)












