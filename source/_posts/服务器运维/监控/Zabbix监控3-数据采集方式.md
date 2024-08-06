---
title: Zabbix监控3-数据采集方式
date: 2023-6-14
tags:
  - Zabbix
  - agent
  - 采集
  - 代理
categories: 
- 运维
- 监控
- zabbix
- 采集/代理
keywords: 'Zabbix,监控,Agent,采集'
cover: https://qiufuqi.github.io/img/hexo/20221116102126.png
abbrlink: zabbix_agent_caiji
url: zabbix_agent_caiji
comments: false
---

**zabbix监控 主动回传 proxy代理**

## agent端主动回传
之前配置都是server端主动采集agent端的数据，此种方式agent端越多zabbix主机的压力就越大，接下来我们让agent端主动将数据发给agent端：(主动被动完全取决于agent端link的模板)

在网页前端，将10.11.8.68上的模板取消连接并清理：
![](https://qiufuqi.github.io/img/hexo/20230614153812.png)
重新选择模板Template OS Linux by Zabbix agent active并更新，此模板是agent端主动将信息回传回来：
![](https://qiufuqi.github.io/img/hexo/20230614153841.png)
可以看到监控项已变更：
![](https://qiufuqi.github.io/img/hexo/20230614154037.png)

### 客户端修改
agent客户端修改，消息采集由proxy来进行。主动模式需要加上端口
``` bash
[root@slave-node ~]# vi /etc/zabbix/zabbix_agentd.conf  #主要修改下面几个参数
StartAgents=0             #客户端agent模式。设置为0表示启用主动模式，而被动模式被关闭，但被监控端的 zabbix_agentd 不监听本地端口
Server=10.11.7.64					#被动模式，指定zabbix-server服务端的ip地址，多个ip的话使用逗号分隔		
ServerActive=10.11.7.64 	#主动模式，指定zabbix-server的ip地址，使用逗号分隔多IP，如果注释这个选项，那么当前服务器的主动监控就被禁用了
Hostname=10.11.8.68       #当使用主动模式时,这个名称的设置必须与zabbix-web监控页面创建的主机名称保持一致

# 以下可用设置
RefreshActiveChecks=180    #被监控端到服务器获取监控项的周期
BufferSize=200              #被监控端存储监控信息的空间大小
Timeout=10                  #超时时间
[root@slave-node ~]# systemctl restart zabbix-agent
```

### 问题处理
#### 问题一：
配置好自动注册后，但Agent注册完成后，Server也可正常接收Agent发送过来的数据，但是可用性一直处于灰色，无法变绿
![](https://qiufuqi.github.io/img/hexo/20230615093948.png)

更改方式一：全局更改
改配置如下：配置 => 模板 — 名称：Template OS Linux by Zabbix agent active
需为主机添加一个Zabbix客户端式监控项 点击 监控项 => 类型选择：Zabbix客户端，选中System local time，启用
强制更新缓存(参考本页最下面)，可用性立马变成绿色

更改方式二：单主机更改
配置 => 主机 => 选中主机，点击监控项 => 类型选择：Zabbix客户端，选中System local time，启用

#### 问题二：
客户端刚变更为active主动时，会有报错信息:
vfs.dev.write.await[sda]" became not supported: Cannot evaluate expression: "Cannot evaluate function "last()": not enough data."
这是因为进行calculated，必须先有数据，才能进行计算，不然的话，可能无法计算，导致出错。
更改配置如下：配置 => 模板 — 名称：Template OS Linux by Zabbix agent active
点击 监控项 => Zabbix客户端(主动式)，选中所有名称，先停用，再启用。
![](https://qiufuqi.github.io/img/hexo/20230615093223.png)
``` bash
[root@zabbix-server ~]# tail -f /var/log/zabbix/zabbix_server.log
16802:20230615:092101.658 item "10.11.8.68:vfs.dev.write.await[sda]" became not supported: Cannot evaluate expression: "Cannot evaluate function "last()": not enough data.".
 16804:20230615:092401.367 item "10.11.8.68:vfs.dev.read.await[sda]" became supported
```


## proxy代理
提高了sever端的效率，但是server端就一个，我们可以通过添加一个proxy代理来进一步减轻server端的压力。
我这里提前安装好了zabbix-proxy ，具体[安装步骤参考](/zabbix_install)

### 创建代理
中文：管理 => agent代理程序 => 创建代理 (右上角) ，**agent代理程序名称要和配置 /etc/zabbix/zabbix_proxy.conf 的Hostname一致，否则会报错**
``` bash
[root@zabbix-proxy ~]# vi /etc/zabbix/zabbix_proxy.conf
DBPassword=proxy-64

# 或者 修改为下列参数
DBName=zabbix
DBUser=zabbix
DBPassword=proxy-64
Server=10.11.7.63     # Server端的地址 agent可以直接向Server发送，也可以向Proxy发送
Hostname=Zabbix proxy
```
![](https://qiufuqi.github.io/img/hexo/20230614154351.png)

### 客户端修改
agent客户端修改，消息采集由proxy来进行。主动模式需要加上端口
``` bash
[root@slave-node ~]# vi /etc/zabbix/zabbix_agentd.conf  #主要修改下面几个参数
StartAgents=0             #客户端agent模式。设置为0表示启用主动模式，而被动模式被关闭，但被监控端的 zabbix_agentd 不监听本地端口
Server=10.11.7.64					#被动模式，指定zabbix-server服务端的ip地址，多个ip的话使用逗号分隔		
ServerActive=10.11.7.64   #主动模式，指定zabbix-server的ip地址，使用逗号分隔多IP，如果注释这个选项，那么当前服务器的主动监控就被禁用了
Hostname=10.11.8.68      #当使用主动模式时,这个名称的设置必须与zabbix-web监控页面创建的主机名称保持一致

# 以下可用设置
RefreshActiveChecks=180    #被监控端到服务器获取监控项的周期
BufferSize=200              #被监控端存储监控信息的空间大小
Timeout=10                  #超时时间
[root@slave-node ~]# systemctl restart zabbix-agent
```
Web端修改代理程序监测
![](https://qiufuqi.github.io/img/hexo/20230614154826.png)


## 强制更新缓存
``` bash
# 强制更新zabbix_server缓存
zabbix_server -R config_cache_reload

# 强制更新zabbix proxy缓存
zabbix_proxy -c /etc/zabbix/zabbix_proxy.conf --runtime-control config_cache_reload
```