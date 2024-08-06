---
title: Zabbix监控2-Agent客户端
date: 2022-11-08
tags:
  - Zabbix
  - Agent
categories: 
- 运维
- 监控
- zabbix
- Agent
keywords: 'Zabbix,监控,Agent'
cover: https://qiufuqi.github.io/img/hexo/20221116102126.png
abbrlink: zabbix_agent
url: zabbix_agent
comments: false
---

**Zabbix监控客户端：window版本 和 linux版本**
zabbix服务器：10.11.7.60
zabbix agent官网[下载地址](https://www.zabbix.com/cn/download_agents?version=5.0+LTS&release=5.0.16&os=Windows&os_version=Any&hardware=amd64&encryption=OpenSSL&packaging=MSI&show_legacy=0#tab:44)

# Window版本
源码压缩包安装，安装包安装，window服务器：10.128.2.200
## Window压缩包
### 下载window安装包
下载对应window版本安装包
链接: https://pan.baidu.com/s/1iN9ie7EkeVEELU_N14bS_Q 提取码: mcte 

### 上传至服务器
1、将压缩包上传至目标服务器
2、新建一个文件夹zabbix（英文路径），将压缩包解压至zabbix文件夹中
3、解压文件共两个子文件夹：bin   conf
4、进入conf文件夹，修改配置文件 zabbix_agentd.conf：
``` bash
# 23行 日志路径写自己解压的路径，或者自己定义一个路径，此路径存放zabbix的日志
LogFile=D:\zabbix\zabbix_agentd.log  
# 106行 zabbix服务器地址
Server=10.11.7.60
# 147行 zabbix服务器地址
ServerActive=10.11.7.60
# 158行 window服务器地址
Hostname=10.128.2.200
```
5、以管理员身份运行命令行，下面操作在命令行执行
``` bash
# 路径是自己解压的路径  bin文件夹
cd C:\zabbix\bin

# 安装客户端  路径是自己解压的路径  conf文件夹的路径
C:\zabbix\bin> zabbix_agentd.exe -i -c C:\zabbix\conf\zabbix_agentd.conf
zabbix_agentd.exe [4808]: service [Zabbix Agent] installed successfully
zabbix_agentd.exe [4808]: event source [Zabbix Agent] installed successfully

# 启动服务
C:\zabbix\bin> zabbix_agentd.exe -s -c C:\zabbix\conf\zabbix_agentd.conf
zabbix_agentd.exe [4748]: service [Zabbix Agent] started successfully
```
![](https://qiufuqi.github.io/img/hexo/20221108140924.png)

### 错误处理
上述操作出现错误则卸载重新安装
``` bash
cd C:\zabbix\bin
C:\zabbix\bin> zabbix_agentd.exe -d -c C:\zabbix\conf\zabbix_agentd.conf
zabbix_agentd.exe [4860]: service [Zabbix Agent] uninstalled successfully
zabbix_agentd.exe [4860]: event source [Zabbix Agent] uninstalled successfully

参数说明：
  -c：指定配置文件所有位置
  -i：安装客户端
  -s：启动客户端
  -x：停止客户端
  -d：卸载客户端
```
### 防火墙端口放行
打开window防火墙：控制面板\系统和安全\Windows 防火墙，点击：允许应用或功能通过windows防火墙
![](https://qiufuqi.github.io/img/hexo/20221108141534.png)
点击：允许其他应用--浏览 找到制定应用
![](https://qiufuqi.github.io/img/hexo/20221108141604.png)
![](https://qiufuqi.github.io/img/hexo/20221108141702.png)

## 安装包安装
官网[下载地址](https://www.zabbix.com/cn/download_agents?version=5.0+LTS&release=5.0.16&os=Windows&os_version=Any&hardware=amd64&encryption=OpenSSL&packaging=MSI&show_legacy=0#tab:44)

### 双机安装包
![](https://qiufuqi.github.io/img/hexo/20221108150142.png)
### 配置zabbix-server参数信息
![](https://qiufuqi.github.io/img/hexo/20221108150221.png)
### 选择zabbix-agent安装目录
默认安装目录C:\Program Files\Zabbix Agent\，需要记住次安装目录，手动启动时需要进入此目录启动客户端。
![](https://qiufuqi.github.io/img/hexo/20221108150327.png)
### 启动zabbix-agent客户端
进入指定目录下C:\Program Files\Zabbix Agent\ ，双机zabbix_agentd.exe

# Linux版本
rpm包安装、源码安装， 推荐使用rpm安装（yum）
## RPM安装
rpm[下载地址](https://repo.zabbix.com/zabbix/5.0/rhel/7/x86_64/)，选择对应版本。
防火墙关闭或者放行端口 zabbix-agent默认端口为10050，zabbix-server默认端口是10051
### 安装zabbix-agent
``` bash
#会默认创建zabbix用户
[root@slave-node ~]# rpm -Uvh https://repo.zabbix.com/zabbix/5.0/rhel/7/x86_64/zabbix-agent-5.0.16-1.el7.x86_64.rpm
[root@slave-node ~]# yum clean all
[root@slave-node ~]# yum -y install zabbix-agent              #查看版本是5.0.16，与我安装的zabbix服务器版本一致
[root@slave-node ~]# systemctl start zabbix-agent							#启动zabbix-agent
[root@slave-node ~]# systemctl enable zabbix-agent						#开机自启zabbix-agent
[root@slave-node ~]# lsof -i:10050												    #zabbix-agent默认端口为10050，zabbix-server默认端口是10051
[root@slave-node ~]# netstat -anp|grep zabbix
tcp        0      0 0.0.0.0:10050           0.0.0.0:*               LISTEN      18888/zabbix_agentd 
tcp6       0      0 :::10050                :::*                    LISTEN      18888/zabbix_agentd
[root@slave-node ~]# tail -f /var/log/zabbix/zabbix_agentd.log #查看zabbix-agent是否启动报错，能不能正常与zabbix-server端通信
```
### 配置zabbix-agent
``` bash
# zabbix-agent的配置文件路径位于/etc/zabbix/zabbix_agentd.conf
[root@slave-node ~]# rpm -qa| grep zabbix*
zabbix-agent-5.0.16-1.el7.x86_64
[root@slave-node ~]# rpm -ql zabbix-agent-5.0.16-1.el7.x86_64
/etc/logrotate.d/zabbix-agent
/etc/zabbix/zabbix_agentd.conf
/etc/zabbix/zabbix_agentd.d
·········
/var/run/zabbix

[root@slave-node ~]# vi /etc/zabbix/zabbix_agentd.conf  #主要修改下面几个参数
Server=10.11.7.60					#被动模式，指定zabbix-server服务端的ip地址，多个ip的话使用逗号分隔		
ServerActive=10.11.7.60:10051		#主动模式，指定zabbix-server的ip地址，使用逗号分隔多IP，如果注释这个选项，那么当前服务器的主动监控就被禁用了
Hostname=10.11.7.232      #当使用主动模式时,这个名称的设置必须与zabbix-web监控页面创建的主机名称保持一致

[root@slave-node ~]# systemctl restart zabbix-agent
```
## 安装包安装
不建议使用这种方式，可能会遇到缺失组件问题。
源码[下载地址](https://www.zabbix.com/cn/download_agents?version=5.0+LTS&release=5.0.16&os=Linux&os_version=4.12&hardware=ppc64le&encryption=No+encryption&packaging=Archive&show_legacy=0#tab:44)
### 下载源码
解压之后，出现下面三个目录:bin,conf,sbin
``` bash
[root@master-node ~]# mkdir /opt/zabbix && cd /opt/zabbix
[root@master-node ~]# wget https://cdn.zabbix.com/zabbix/binaries/stable/5.0/5.0.16/zabbix_agent-5.0.16-linux-3.0-amd64-static.tar.gz
[root@master-node ~]# tar -zxvf zabbix_agent-5.0.16-linux-3.0-amd64-static.tar.gz
```
### 创建用户和组
新建zabbix用户并将其加入到zabbix组，并将他设置为不可登录的类型的用户。
``` bash
[root@master-node zabbix_agent]# groupadd zabbix
[root@master-node zabbix_agent]# useradd -g zabbix zabbix -s /sbin/nologin
```
### 进入bin目录
``` bash
[root@master-node zabbix_agent]# cd bin && ls
zabbix_get  zabbix_sender
[root@master-node zabbix_agent]# cp /opt/zabbix/sbin/zabbix_sender /opt/zabbix/sbin/zabbix_get /usr/bin/
```
### 进入sbin目录
可执行文件是zabbix_agentd的客户端的可执行文件
``` bash
[root@master-node bin]# cd ../sbin/ && ls
zabbix_agentd
[root@master-node sbin]# cp /opt/zabbix/sbin/zabbix_agentd /usr/sbin/
```
### 进入conf目录
zabbix_agentd.conf，是zabbix-agent的配置文件
``` bash
[root@master-node sbin]# cd ../conf/ && ls
zabbix_agentd  zabbix_agentd.conf
[root@master-node conf]# cp /opt/zabbix/conf/zabbix_agentd.conf  /usr/local/etc/
```
### 修改配置文件
``` bash
[root@master-node conf]# vi /usr/local/etc/zabbix_agentd.conf
LogFile=/var/log/zabbix/zabbix_agentd.log
Server=10.11.7.60					#被动模式，指定zabbix-server服务端的ip地址，多个ip的话使用逗号分隔		
# ServerActive=10.11.7.60		#主动模式，指定zabbix-server的ip地址，使用逗号分隔多IP，如果注释这个选项，那么当前服务器的主动监控就被禁用了
Hostname=10.11.7.232      #当使用主动模式时,这个名称的设置必须与zabbix-web监控页面创建的主机名称保持一致
```
### 创建日志文件
``` bash
[root@master-node conf]# mkdir /var/log/zabbix/
[root@master-node conf]# chown zabbix:zabbix /var/log/zabbix/
[root@master-node conf]# chmod 777 /var/log/zabbix/
[root@master-node conf]# touch  /var/log/zabbix/zabbix_agentd.log
[root@master-node conf]# chmod 777 /var/log/zabbix/zabbix_agentd.log
```
### 添加监控端口
如果存在，则无需处理
``` bash
[root@master-node conf]# vi /etc/services
zabbix-agent 10050/tcp
zabbix-agent 10050/udp
```
### 启动服务
``` bash
[root@master-node ~]# cp /opt/zabbix/sbin/zabbix_agentd /etc/init.d/
# 启动服务 
[root@master-node ~]# /etc/init.d/zabbix_agentd

# 服务可能没有启动起来  没有权限 建立zabbix_agentd.pid并赋予权限
[root@master-node ~]# ps -ef | grep zabbix  
[root@localhost sbin]# touch  /tmp/zabbix_agentd.pid
[root@localhost sbin]# chmod 777 /tmp/zabbix_agentd.pid
[root@master-node ~]# /etc/init.d/zabbix_agentd     # 已正常启动
```

# 添加zabbix监控
## window添加监控
客户机：10.128.2.200
zabbix服务器端，点击左侧边栏，配置----主机----右上角的创建主机；
``` bash
主机名称：10.128.2.200 写IP地址，便于分辨，此处填写内容与下文中158行内容需保持一致
可见的名称：10.128.2.200  可写可不写
群组：Templates/Operating systems;
interfaces：类型客户端，IP 10.128.2.200，端口10050；
描述：可加可不加
其他不动

模板：Template OS Windows by Zabbix agent
其他的不动
```
主机：
![](https://qiufuqi.github.io/img/hexo/20230614152432.png)
模板：
![](https://qiufuqi.github.io/img/hexo/20230614152444.png)
创建完成后可看见ZBX绿色图标
![](https://qiufuqi.github.io/img/hexo/20221108142656.png)

## linux添加监控
客户机：10.11.7.232
zabbix服务器端，点击左侧边栏，配置----主机----右上角的创建主机；
``` bash
主机名称：10.11.7.232 写IP地址，便于分辨，此处填写内容与下文中158行内容需保持一致
可见的名称：10.11.7.232  可写可不写
群组：Templates/Operating systems;
interfaces：类型客户端，IP 10.11.7.232，端口10050；
描述：可加可不加
其他不动

模板：Template OS Linux by Zabbix agent
其他的不动
```
主机：
![](https://qiufuqi.github.io/img/hexo/20230614152432.png)
模板：
![](https://qiufuqi.github.io/img/hexo/20230614152444.png)
创建完成后可看见ZBX绿色图标
![](https://qiufuqi.github.io/img/hexo/20221115171250.png)


