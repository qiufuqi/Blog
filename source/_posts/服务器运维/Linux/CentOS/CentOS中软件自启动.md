---
title: CentOS中软件自启动三种方式
date: 2022-08-14 15:25:50
tags:
  - Linux
  - CentOS
  - 自启动
categories: 
- 运维
- 自启动
keywords: 'Linux,CentOS,自启动'
description: CentOS软件自启动方式
cover: https://qiufuqi.github.io/img/hexo/20231205140400.png
abbrlink: centos_autostart
comments: false
---

Linux环境中软件自启动的三种方式，系统版本CentOS 7.6。

## 介绍
1、systemd服务\
2、使用 /etc/rc.d/rc.local\
3、使用crontab定时计划中的@reboot

## 操作

### systemd服务
参考示例：

#### 创建自启动文件
``` bash
vi /usr/lib/systemd/system/alertmanager.service
```

#### 编辑写入启动配置
``` bash
[Unit]
Description=https://prometheus.io
 
[Service]
Restart=on-failure
ExecStart=/usr/local/alertmanager/alertmanager --config.file=/usr/local/alertmanager/alertmanager.yml
[Install]
WantedBy=multi-user.target
```
#### 设置服务生效 && 自启动
``` bash
systemctl daemon-reload
systemctl start alertmanager.service
systemctl enable alertmanager.service
```
#### 取消自启动
``` bash
systemctl disable alertmanager.service

systemctl is-enable alertmanager   #是否开机启动
systemctl is-active alertmanager   #是否启动状态

```

### rc.local方式

#### 打开rc.local
``` bash
vi /etc/rc.d/rc.local
```
#### 添加执行内容
``` bash
/home/shell/init.sh
```
#### 修改rc.local权限
``` bash
chmod +x /etc/rc.d/rc.local
```
#### 取消方式
打开/etc/rc.d/rc.local文件，删除执行内容

### crontab定时计划中的@reboot

#### 打开定时脚本
``` bash
crontab -e
```
#### 添加定时任务
``` bash
@reboot /home/test.sh
```
#### 取消方法
``` bash
crontab -e
删除定时任务
```