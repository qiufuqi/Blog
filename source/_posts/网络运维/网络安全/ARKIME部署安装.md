---
title: ARKIME部署安装
date: 2024-8-5
tags:
  - 网络安全
  - arkime
categories: 
- 运维
- 网络安全
- arkime
keywords: '网络安全,arkime'
cover: https://qiufuqi.github.io/img/hexo/20240805144344.png
abbrlink: arkime_install
url: arkime_install
comments: false
---
[文章参考](https://blog.csdn.net/lhyzws/article/details/124433680)
[文章参考](https://blog.csdn.net/haoyunbin/article/details/131050113)
# 什么是Arkime

Arkime（以前叫Moloch）是一个大规模的开源索引数据包捕获和搜索系统。
Arkime增强了您当前的安全基础设施，以标准PCAP格式存储和索引网络流量，提供快速的索引访问。为PCAP浏览、搜索和导出提供了直观简单的web界面。Arkime公开了API，允许直接下载和使用PCAP数据和JSON格式的会话数据。Arkime以标准PCAP格式存储和导出所有数据包，允许您在分析工作流程中使用最喜欢的PCAP摄取工具，如wireshark。
Arkime可以跨多个系统部署，可以扩展到处理每秒数十GB的流量。PCAP保留基于可用的传感器磁盘空间。元数据保留基于Elasticsearch集群规模。两者都可以随时增加，并在您的完全控制下。

# 下载和安装
[下载arkime安装包](https://github.com/arkime/arkime/releases/)
根据系统版本下载对应版本，对于Centos 8或RHEL 8使用el8下载，而不是RHEL 9。
## 安装依赖
arkime安装主要就3个依赖，分别是perl-JSON、perl-libwww-perl和perl-LWP-Protocol-https。使用yumdownloader解析并下载依赖包后安装。
![](https://qiufuqi.github.io/img/hexo/20240805154909.png)
```bash
yumdownloader --resolve perl-JSON、perl-libwww-perl perl-LWP-Protocol-https
yum -y install perl-JSON、perl-libwww-perl perl-LWP-Protocol-https
```
## 安装arkime
使用rpm安装下载的rpm文件
``` bash
rpm -ivh arkime-main-el8.x86_64.rpm
```
## 安装elastic
打开/opt/arkime/bin/Configure脚本，找到如下安装ES的这一行，手动下载该rpm并安装
``` bash

```
## 配置
安装好后执行/opt/arkime/bin/Configure配置工具进行后续配置，geo数据库建议选no。
![](https://qiufuqi.github.io/img/hexo/20240805155048.png)
依次按照步骤执行提示



