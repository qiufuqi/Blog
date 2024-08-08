---
title: Suricata部署安装
date: 2024-8-8
tags:
  - 网络安全
  - Suricata
categories: 
- 运维
- 网络安全
- Suricata
keywords: '网络安全,Suricata'
cover: https://qiufuqi.github.io/img/hexo/20240808094629.png
abbrlink: suricata_install
url: suricata_install
comments: false
---

[中文参考文档](https://www.osgeo.cn/suricata/what-is-suricata.html)[英文参考文档](https://docs.suricata.io/en/suricata-7.0.6/) [官方网站](https://suricata.io/download/)

# 介绍
Suricata 是一个高性能的网络入侵检测和防御系统（IDS/IPS）。它是由OISF开发，完全开源，并且可以免费使用。Suricata是一个开源的IDS，能够实时监控网络流量，检测和防御潜在的威胁。
Suricata的设计注重性能和可扩展性，它可以在低至中等规格的硬件上运行，支持多线程，同时处理高吞吐量的网络流量，同时流分析功能更为强大和复杂。
Suricata能够检测各种网络威胁，包括但不限于：
- 协议解析：支持多种协议，如TCP, UDP, ICMP, HTTP, FTP等。
- 签名匹配：使用类似于Snort的规则语言进行签名匹配，以检测已知攻击模式。
- 异常检测：可以检测到异常行为，如异常流量或潜在的恶意行为。
- 应用层检测：能够检测应用层的攻击和异常行为，例如SQL注入攻击。
- 流量分析：可以对网络流量进行深入分析，包括状态跟踪和流量重建。
- 实时响应：在检测到攻击时，可以实时阻断或记录攻击流量。

Suricata的一些主要运行模式：
- Single 模式：在这种模式下，所有的数据处理任务都由单个工作线程完成。
- Workers 模式：这是为了高性能而设计的模式。在Workers模式中，每个工作线程都独立执行从数据包捕获到日志记录的所有任务，以实现负载均衡和提高处理速度。
- Autofp 模式：这种模式适用于处理PCAP文件或在某些IPS设置的情况下。Autofp模式下，有一个或多个数据包捕获线程，它们捕获数据包并进行解码，然后将数据包传递给 flow worker 线程。

# 安装
当前测试环境CentOS8，当前Suricate的最新版本为7 [安装参考步骤](https://docs.suricata.io/en/suricata-7.0.6/install.html#rpm-packages)
## 源码安装
从源分发文件安装可以最大程度地控制Suricata安装。

提前安装依赖 gcc pcre2-devel libyaml-devel jansson-devel libpcap-devel
``` bash
yum -y install gcc pcre2-devel jansson-devel libpcap-devel
wget https://pyyaml.org/download/libyaml/yaml-0.1.4.tar.gz
tar xzvf yaml-0.1.4.tar.gz
cd yaml-0.1.4
./configure --prefix=/usr/local    #注意此处勿改路径！否则库文件无法写入正确目录
make && make install
```



安装Suricate
``` bash
wget https://www.openinfosecfoundation.org/download/suricata-7.0.6.tar.gz

tar xzvf suricata-7.0.6.tar.gz
cd suricata-7.0.6
./configure
make
make install
```
## yum 安装
yum安装有可能不成功
``` bash
yum -y install epel-release yum-plugin-copr
yum -y copr enable @oisf/suricata-7.0
yum -y install suricata
```