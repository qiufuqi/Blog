---
title: Zabbix监控1-Server和Proxy
date: 2023-06-13
tags:
  - Zabbix
  - Server
  - Proxy
categories: 
- 运维
- 监控
- zabbix
- server/proxy
keywords: 'Zabbix,监控,server/proxy'
cover: https://qiufuqi.github.io/img/hexo/20221116102126.png
abbrlink: zabbix_install
url: zabbix_install
comments: false
---

# Zabbix简介
[zabbix官网](https://www.zabbix.com/)
zabbix是一个基于WEB界面的提供分布式系统监视以及网络监视功能的企业级的开源解决方案。
zabbix能监视各种网络参数，保证服务器系统的安全运营；并提供灵活的通知机制以让系统管理员快速定位/解决存在的各种问题。
zabbix由2部分构成，zabbix server与可选组件zabbix agent。
zabbix server可以通过SNMP，zabbix agent，ping，端口监视等方法提供对远程服务器/网络状态的监视，数据收集等功能，它可以运行在Linux，Solaris，HP-UX，AIX，Free BSD，Open BSD，OS X等平台上。
zabbix主要做到以下监控：
- CPU负荷
- 内存使用
- 磁盘使用
- 网络状况
- 端口监视
- 日志监视

# 环境搭建
提前环境准备，参考zabbix官网来，我的环境用的是CentOS7.6 ，Zabbix-server用的是5.0 LTS Nginx版本
``` bash
Zabbix Server 10.11.7.63
Zabbix Proxy  10.11.7.64/10.11.8.99
```
![](https://qiufuqi.github.io/img/hexo/20230613145108.png)
这里zabbix server和zabbix proxy是同一个网段下，zabbix proxy有两块网卡，一个是7，一个是8网段 ，8网段的agent数据收集通过proxy代理进行收集信息，在一定时间内，批量上传至server，这样可以避免频繁访问server端，对服务器造成压力
## zabbix-server安装
本次选择zabbix-server 5.0 LTS Nginx版本。[官网地址](https://www.zabbix.com/cn/download?zabbix=5.0&os_distribution=centos&os_version=7&components=server_frontend_agent&db=mysql&ws=nginx)
官网推荐通过二进制包来安装,**具体命令参考官网地址即可**
![](https://qiufuqi.github.io/img/hexo/20230613153000.png)

### 基础配置
关闭防火墙，selinux，安装依赖包等
``` bash
[root@zabbix-server ~]# systemctl disable firewalld
[root@zabbix-server ~]# vi /etc/sysconfig/selinux
[root@zabbix-server ~]# yum -y install epel-release net-tools
```
### 获取zabbix仓库
``` bash
[root@zabbix-server ~]# rpm -Uvh https://repo.zabbix.com/zabbix/5.0/rhel/7/x86_64/zabbix-release-5.0-1.el7.noarch.rpm
[root@zabbix-server ~]# yum clean all
```
### 安装zabbix-server
安装Zabbix server，Web前端，agent
``` bash
[root@zabbix-server ~]# yum -y install zabbix-server-mysql zabbix-agent
```
### 安装zabbix前端包
安装zabbix前端包，编辑配置文件 /etc/yum.repos.d/zabbix.repo ，确保zabbix-frontend repository.
``` bash
[root@zabbix-server ~]# yum -y install centos-release-scl
[root@zabbix-server ~]# vi /etc/yum.repos.d/zabbix.repo
·········
[zabbix-frontend]
·········
enabled=1
·········
[root@zabbix-server ~]# yum -y install zabbix-web-mysql-scl zabbix-nginx-conf-scl
```
### 数据库操作
我这里选用的是Mysql数据库，数据库可以是本机安装，也可以是其他现有的数据库（配置zabbix-server时数据库参数对应修改）。确保Mysql正确安装且运行，[数据库安装参考。](/centos_mysql5.7)
执行以下代码，创建zabbix数据库，并创建zabbix账户，密码是：server-63
``` bash
[root@zabbix-server ~]# mysql -uroot -p
password
mysql> create database zabbix character set utf8 collate utf8_bin;
mysql> create user zabbix@localhost identified by 'server-63';
mysql> grant all privileges on zabbix.* to zabbix@localhost;
mysql> set global log_bin_trust_function_creators = 1;
mysql> quit;
```
导入初始架构和数据，系统将提示您输入新创建的密码。执行成功后关闭mysql的 log_bin_trust_function_creators 参数
``` bash
[root@zabbix-server ~]# zcat /usr/share/doc/zabbix-server-mysql*/create.sql.gz | mysql -uzabbix -p zabbix

[root@zabbix-server ~]# mysql -uroot -p
password
mysql> set global log_bin_trust_function_creators = 0;
mysql> quit;
```
### 配置zabbix-server数据库
编辑配置文件 /etc/zabbix/zabbix_server.conf，大概124行
如果上一步的数据库不是安装在本机，则修改对应的参数（DBHost，DBName）
``` bash
[root@zabbix-server ~]# vi /etc/zabbix/zabbix_server.conf
DBPassword=server-63

# 或者修改为
DBName=zabbix
DBUser=zabbix
DBPassword=server-63
Server=10.11.7.63
Hostname=zabbix-server
```
### 配置zabbix的前端PHP
编辑配置文件 /etc/opt/rh/rh-nginx116/nginx/conf.d/zabbix.conf，设置端口和域名（自己指定，不一定一样）
``` bash
[root@zabbix-server ~]# vi /etc/opt/rh/rh-nginx116/nginx/conf.d/zabbix.conf
listen 80;
server_name zabbix.yurun.com;
```
编辑配置文件 /etc/opt/rh/rh-php72/php-fpm.d/zabbix.conf，设置acl_users 
``` bash
[root@zabbix-server ~]# vi /etc/opt/rh/rh-php72/php-fpm.d/zabbix.conf
·····
listen.acl_users = apache,nginx
php_value[date.timezone] = Asia/Shanghai
```
### 启动zabbix-server和agent进程
启动Zabbix server和agent进程，并为它们设置开机自启，无法启动时参考最后的**启动问题处理。**
``` bash
[root@zabbix-server ~]# systemctl restart zabbix-server zabbix-agent rh-nginx116-nginx rh-php72-php-fpm
[root@zabbix-server ~]# systemctl enable zabbix-server zabbix-agent rh-nginx116-nginx rh-php72-php-fpm

# 数据库自启动，如果数据库在同一台服务器
[root@zabbix-server ~]# systemctl enable mysqld
```
### 访问zabbix
通过设置的域名访问zabbix：http://zabbix.yurun.com/setup.php，如果没设置就直接用IP访问，按照步骤完成后续安装。
![](https://qiufuqi.github.io/img/hexo/20230613164415.png)
完成后默认登录账号：Admin 密码：zabbix


## zabbix-proxy安装
zabbix proxy可以代替zabbix server检索客户端的数据，然后把数据汇报给zabbix server，并且在一定程度上分担了zabbix server的压力.zabbix proxy可以非常简便的实现了集中式、分布式监控。[安装参考文档](https://www.zabbix.com/cn/download?zabbix=5.0&os_distribution=centos&os_version=7&components=proxy&db=mysql&ws=)和zabbix-server版本要统一。
zabbix-proxy使用场景:
- 监控远程区域设备
- 监控本地网络不稳定区域
- 当zabbix监控上千设备时，使用它来减轻server的压力
- 简化zabbix的维护

### 基础配置
关闭防火墙，selinux，安装依赖包等
``` bash
[root@zabbix-proxy ~]# systemctl disable firewalld
[root@zabbix-proxy ~]# vi /etc/sysconfig/selinux
[root@zabbix-proxy ~]# yum -y install epel-release net-tools
```
### 获取zabbix仓库
``` bash
[root@zabbix-proxy ~]# rpm -Uvh https://repo.zabbix.com/zabbix/5.0/rhel/7/x86_64/zabbix-release-5.0-1.el7.noarch.rpm
[root@zabbix-proxy ~]# yum clean all
```
### 安装zabbix-proxy
安装Zabbix proxy，Web前端，agent
``` bash
[root@zabbix-proxy ~]# yum -y install zabbix-proxy-mysql
```
### 数据库操作
我这里选用的是Mysql数据库，数据库可以是本机安装，也可以是其他现有的数据库（配置zabbix-proxy时数据库参数对应修改）。确保Mysql正确安装且运行，[数据库安装参考。](/centos_mysql5.7)
执行以下代码，创建zabbix数据库，并创建zabbix账户，密码是：proxy-64
``` bash
[root@zabbix-proxy ~]# mysql -uroot -p
password
mysql> create database zabbix_proxy character set utf8 collate utf8_bin;
mysql> create user zabbix@localhost identified by 'proxy-64';
mysql> grant all privileges on zabbix_proxy.* to zabbix@localhost;
mysql> set global log_bin_trust_function_creators = 1;
mysql> quit;
```
导入初始架构和数据，系统将提示您输入新创建的密码。执行成功后关闭mysql的 log_bin_trust_function_creators 参数
``` bash
[root@zabbix-proxy ~]# zcat /usr/share/doc/zabbix-proxy-mysql*/schema.sql.gz | mysql -uzabbix -p zabbix_proxy

[root@zabbix-proxy ~]# mysql -uroot -p
password
mysql> set global log_bin_trust_function_creators = 0;
mysql> quit;
```
### 配置zabbix-proxy数据库
编辑配置文件 /etc/zabbix/zabbix_proxy.conf，大概196行
如果上一步的数据库不是安装在本机，则修改对应的参数（DBHost，DBName）
``` bash
[root@zabbix-proxy ~]# vi /etc/zabbix/zabbix_proxy.conf
DBPassword=proxy-64

# 或者 修改为下列参数
DBName=zabbix
DBUser=zabbix
DBPassword=proxy-64
Server=10.11.7.63     # proxy向server发送数据
Hostname=Zabbix proxy
```
### 启动zabbix-proxy进程
启动Zabbix proxy并为它们设置开机自启，无法启动时参考最后的**启动问题处理。**
``` bash
[root@zabbix-proxy ~]# systemctl restart zabbix-proxy
[root@zabbix-proxy ~]# systemctl enable zabbix-proxy

# 数据库自启动，如果数据库在同一台服务器
[root@zabbix-proxy ~]# systemctl enable mysqld
```
## zabbix配置代理
登录到zabbix-server网址后，设置代理，路径如下：
Administration => Proxies => Create proxy (右上角)，填写zabbix-proxy的相关信息。
注意：**agent代理程序名称要和配置 /etc/zabbix/zabbix_proxy.conf 的Hostname一致，否则会报错**
![](https://qiufuqi.github.io/img/hexo/20230614091213.png)
![](https://qiufuqi.github.io/img/hexo/20230614091444.png)

中文：管理 => agent代理程序 => 创建代理 (右上角) ，代理名称要和proxy.conf配置里的hostname一致。
![](https://qiufuqi.github.io/img/hexo/20230614140645.png)
``` bash
[root@zabbix-proxy ~]# tail -f /var/log/zabbix/zabbix_proxy.log 
 27095:20230614:111711.813 cannot send proxy data to server at "10.11.7.63": proxy "Zabbix proxy" not found
 27095:20230614:111712.814 cannot send proxy data to server at "10.11.7.63": proxy "Zabbix proxy" not found
```

## zabbix-agent安装
zabbix-agent多端，linux或者window都可以，[安装参考。](/zabbix_agent)，端口一般10050，可以关闭防火墙或者放行。
zabbix-agent监控方式可分为主动和被动
在zabbix web页面Monitoring->Configuration->Hosts 页面更改Host name和zabbix_agentd.conf里面的Hostname一样。(建议直接写IP) 
``` bash
[root@zabbix-proxy ~]# vi /etc/zabbix/zabbix_agentd.conf
Server=10.11.8.64
ServerActive=10.11.8.64
Hostname=10.11.8.68   # 或者当前被监控IP

[root@zabbix-proxy ~]# systemctl restart zabbix-agent
```


# 强制更新缓存
``` bash
# 强制更新zabbix_server缓存
zabbix_server -R config_cache_reload

# 强制更新zabbix proxy缓存
zabbix_proxy -c /etc/zabbix/zabbix_proxy.conf --runtime-control config_cache_reload
```
# 启动问题处理
zabbix-server，zabbix-proxy无法启动问题，发现缺失libmysqlclient.so.18，[处理参考](https://blog.csdn.net/LT_Future/article/details/103648662)
``` bash
[root@zabbix-proxy ~]#  journalctl -xe
·········
Jun 13 16:26:35 zabbix-server zabbix_server[16346]: /usr/sbin/zabbix_server: error while loading shared libraries: libmysqlclient.so.18: cannot open shared obj
·········
```
错误原因：安装mysql5.7时，手动卸载了系统自带的mariadb-libs-5.5.56-2.el7.x86_64，安装mysql之后只有libmysqlclient.so.20，所以找不到libmysqlclient.so.18文件
安装mysql-community-libs-compat-5.7.31-1.el7.x86_64.rpm，这个安装包中会包括所需版本的libmysqlclient.so（跟数据库版本一致），[下载地址](https://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/)
``` bash
[root@zabbix-proxy ~]# wget https://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/mysql-community-libs-compat-5.7.31-1.el7.x86_64.rpm
[root@zabbix-proxy ~]# rpm -ivh mysql-community-libs-compat-5.7.31-1.el7.x86_64.rpm
# 此时缺失的文件已存在
[root@zabbix-proxy ~]# ll /usr/lib64/mysql/
total 15276
lrwxrwxrwx  1 root root      20 Jun 13 16:36 libmysqlclient_r.so.18 -> libmysqlclient.so.18
lrwxrwxrwx  1 root root      24 Jun 13 16:36 libmysqlclient_r.so.18.1.0 -> libmysqlclient.so.18.1.0
lrwxrwxrwx  1 root root      24 Jun 13 16:36 libmysqlclient.so.18 -> libmysqlclient.so.18.1.0
-rwxr-xr-x  1 root root 5983544 Jun  2  2020 libmysqlclient.so.18.1.0
lrwxrwxrwx  1 root root      25 Jun 13 15:49 libmysqlclient.so.20 -> libmysqlclient.so.20.3.18
-rwxr-xr-x  1 root root 9652456 Jun  2  2020 libmysqlclient.so.20.3.18
drwxr-xr-x  4 root root      28 Jun 13 15:52 mecab
drwxr-xr-x. 3 root root    4096 Jun 13 15:52 plugin
```
此时重启zabbix-server或者zabbix-proxy即可