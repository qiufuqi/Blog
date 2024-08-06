---
title: CentOS防火墙管理
date: 2022-08-14 16:13:50
tags:
  - Linux
  - CentOS
  - 防火墙
categories: 
- 运维
- 防火墙
keywords: 'Linux,CentOS,防火墙'
description: CentOS防火墙管理
cover: https://qiufuqi.github.io/img/hexo/20231205140124.png
abbrlink: centos_firewalld
comments: false
---

Linux环境中防火墙管理，系统版本centos7.6。

## 基本命令

### 启动防火墙
``` bash
systemctl start firewalld.service
```

### 开机自启 && 关闭
``` bash
systemctl enable firewalld.service
systemctl disable firewalld.service
```

### 防火墙属性
``` bash
firewall-cmd --list-all
```

### 查看所有端口
``` bash
firewall-cmd --zone=public --list-ports
```

### 查看指定端口
``` bash
firewall-cmd --zone=public --query-port=80/tcp
```

### 添加端口
``` bash
firewall-cmd --zone=public --add-port=80/tcp --permanent 
```

### 删除端口
``` bash
firewall-cmd --zone=public --remove-port=80/tcp --permanent
```

### 拒绝指定IP
``` bash
# 拒绝网段或单个IP访问
firewall-cmd --permanent --zone=block --add-source=172.34.0.0/24
firewall-cmd --permanent --zone=block --add-source=172.34.4.207
firewall-cmd --permanent --zone=block --add-service=http
firewall-cmd --permanent --zone=block --add-service=ftp

firewall-cmd --permanent --zone=block --remove-source=172.34.4.207
```

### keepalived放行
可在destination 前添加 --in-interface ens192
``` bash
firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 --destination 224.0.0.18 --protocol vrrp -j ACCEPT
firewall-cmd --direct --permanent --add-rule ipv4 filter OUTPUT 0 --destination 224.0.0.18 --protocol vrrp -j ACCEPT
```

### 重新加载防火墙
``` bash
firewall-cmd --reload
systemctl reload firewalld.service
```