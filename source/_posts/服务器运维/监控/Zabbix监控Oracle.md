---
title: Zabbix监控Oracle
date: 2022-11-17
tags:
  - Zabbix
  - Oracle
categories: 
- 运维
- 监控
- zabbix
keywords: 'Zabbix,监控'
cover: https://qiufuqi.github.io/img/hexo/20221117170417.png
abbrlink: zabbix_oracle
url: zabbix_oracle
comments: false
---

zabbix是一款非常强大，同时也是应用最为广泛的开源监控软件。

## 监控方式
zabbix监控的方式主要有以下三种类型
### Zabbix agent
在被监控机上面安装zabbix agent，zabbix agent将被监控机采集到的数据发送给zabbix server。这种方式最常用，一般用来采集服务器的cpu、内存等信息。[参考](/zabbix_agent)
### SNMP
一些网络设备如交换机，上面无法安装zabbix agent，所以只能通过snmp的方式收集监控数据如端口状态，流量等。
### External check
在zabbix server上面运行查询脚本，直接查询被监控机上的数据。**此种方式在被监控机上面不需要做任何部署**，所有查询全部从zabbix server上面发出，所以对zabbix server的性能要求较高，官方不推荐大量使用该方式。对于少量的oracle数据库服务器，可以采用该方式。

## 规划监控项
- 数据库空间不足或数据库发生故障，DBA需要立即处理。
监控项包括表空间、用户状态、实例状态、锁、大量等待事件、闪回区使用率等。此类监控项需要给其设置触发器，一旦出现异常，及时告警。

- 数据库运行状态的一些统计信息，为DBA定位数据库性能问题发生的时间和类别提供参考。
监控项包括常见的等待事件发生的次数，命中率、硬解析比例等。


下面表格中列出附件中模板的监控项
![](https://qiufuqi.github.io/img/hexo/20221117085817.png)
![](https://qiufuqi.github.io/img/hexo/20221117085828.png)
![](https://qiufuqi.github.io/img/hexo/20221117085840.png)

## 安装
进入正式安装环节，我假定你已经安装了zabbix server，因此这里略过zabbix server的安装步骤。
**以下所有操作均在zabbix服务器上面执行**
### 安装oracle客户端
从[官网](https://www.oracle.com/cn/database/technologies/instant-client/downloads.html)下载如下三个rpm包；
百度网盘：https://pan.baidu.com/s/1f7jEKpYQ1GviooFrN38zjg 提取码: xsqw
oracle-instantclient11.2-basic-11.2.0.4.0-1.x86_64.rpm
oracle-instantclient11.2-devel-11.2.0.4.0-1.x86_64.rpm
oracle-instantclient11.2-sqlplus-11.2.0.4.0-1.x86_64.rpm
使用root安装oracle客户端
``` bash
[root@zabbix opt]# rpm -ivh oracle-instantclient11.2-basic-11.2.0.4.0-1.x86_64.rpm
[root@zabbix opt]# rpm -ivh oracle-instantclient11.2-devel-11.2.0.4.0-1.x86_64.rpm
[root@zabbix opt]# rpm -ivh oracle-instantclient11.2-sqlplus-11.2.0.4.0-1.x86_64.rpm
```
### 配置环境变量
``` bash
[root@zabbix client64]# vi /etc/profile
·········
export ORACLE_HOME=/usr/lib/oracle/11.2/client64
export LD_LIBRARY_PATH=$ORACLE_HOME/lib
export PATH=$PATH:$ORACLE_HOME/bin
·········
# 使配置生效
[root@zabbix client64]# source /etc/profile
```
### 配置动态库
``` bash
[root@zabbix ld.so.conf.d]# vi /etc/ld.so.conf/oracle.conf
# 使配置生效
[root@zabbix ld.so.conf.d]# ldconfig
```
### oracle进行测试
出现下面的提示证明oracle client安装成功
**sqlplus 用户名/密码@ip:1521/服务名**
``` bash
[root@zabbix ~]# sqlplus orcl1/密码@10.11.7.126:1521/ORCL
SQL*Plus: Release 11.2.0.4.0 Production on Thu Nov 17 09:32:43 2022
Copyright (c) 1982, 2013, Oracle.  All rights reserved.
Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options

SQL>
SQL> exit
Disconnected from Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options
```
![](https://qiufuqi.github.io/img/hexo/20221117093301.png)

### 安装phthon相关包
python升级(/python_upgrate)
查看python版本 python -V
python版本与cx_Oracle版本要一致，我的安装cx_Oracle2.7失败，改用了python3.6

安装cx_Oracle(python连接oracle的包)
``` bash
[root@zabbix ~]# wget http://downloads.sourceforge.net/project/cx-oracle/5.1.2/cx_Oracle-5.1.2-11g-py26-1.x86_64.rpm
[root@zabbix ~]# rpm -ivh cx_Oracle-5.1.2-11g-py26-1.x86_64.rpm
或者
[root@zabbix ~]# pip install cx_Oracle
```
安装argparse,下载3.6版本
``` bash
[root@zabbix ~]# wget https://bootstrap.pypa.io/pip/3.6/get-pip.py --no-check-certificate
[root@zabbix ~]# python get-pip.py
[root@zabbix ~]# pip install argparse
```
### 上传python脚本
下载附件[下载地址](https://github.com/qiufuqi/zabbix_template)
将附件中的pyora.py脚本放入/usr/lib/zabbix/externalscripts/目录下
赋权限，让zabbix用户能够执行该脚本
``` bash
[root@zabbix ~]# cd /usr/lib/zabbix/externalscripts/
[root@zabbix externalscripts]# chmod 755 /usr/lib/zabbix/externalscripts/pyora.py
```
[注意：先在被监控机的oracle数据库中创建监控用户，用户名和密码可以自己随意指定
create user zabbix identified by zabbix;
grant connect, select any dictionary to zabbix;

### 测试脚本
orcl 可进入oracle 执行 lsnrctl status查看
python pyora.py --username zabbix --password zabbix --address 10.11.7.126 --port 1521 --database orcl show_tablespaces
上面测试脚本的参数说明：
- username: 用户名
- password: 密码
- address: 被监控机ip地址
- port: 端口号
- database: orcl show_tablespaces
``` bash
[root@zabbix externalscripts]# python pyora.py --username zabbix --password zabbix --address 10.11.7.126 --port 1521 --database orcl show_tablespaces
{"data": [{"{#TABLESPACE}": "SYSTEM"}, {"{#TABLESPACE}": "SYSAUX"}, {"{#TABLESPACE}": "UNDOTBS1"}, {"{#TABLESPACE}": "USERS"}, {"{#TABLESPACE}": "ECOLOGY"}, {"{#TABLESPACE}": "EXAMPLE"}]}
```
有返回结果表示脚本能正常运行

### 上传模板文件
将附件中的Pyora_ExternalCheck_11G.xml模板导入到zabbix server中
在zabbix页面中，依次点击Configuration – Templates – Import – 选择文件 – Import，即完成了导入
![](https://qiufuqi.github.io/img/hexo/20221117163441.png)

### 添加机器
添加机器，并链接到模板
在zabbix页面中，依次点击Configuration – Hosts – Create host – Hostname (输入ip地址) – groups (选Linux servers) – Templates (选择Pyora_ExternalCheck_11G) – 点击上面的Add – Macros – 点击上面的Add添加宏，全部添加完毕后，点击下面的Add，主机即添加完毕。
![](https://qiufuqi.github.io/img/hexo/20221117164335.png)













https://blog.csdn.net/zfw_666666/article/details/124712980