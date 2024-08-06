---
title: CentOS安装Mysql5.7
date: 2022-08-14 17:45:50
tags:
  - Linux
  - CentOS
  - Mysql
categories: 
- 运维
- 数据库
- Mysql
keywords: 'Linux,CentOS,Mysql'
description: CentOS安装Mysql5.7
cover: https://qiufuqi.github.io/img/hexo/20231205135528.png
abbrlink: centos_mysql5.7
comments: false
---

Linux环境中安装Mysql5.7，系统版本centos7.6。

**Mysql5.7安装**
本文总共包含4种安装方式，均通过验证，推荐使用yum安装（最新版本），rpm可安装指定版本，安装包可安装指定版本。
安装方式如下：
- YUM安装
- RPM安装
- 安装包安装
- [docker安装](/docker_mysql)

## YUM安装
Mysql5.7 YUM[下载地址](https://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/)

### 卸载之前版本

``` bash 
# rpm -e +安装包 或  yum remove +安装包
# rpm -ev --nodeps  +安装包  强制删除

[root@localhost ~]# rpm -qa|grep mariadb
mariadb-libs-5.5.68-1.el7.x86_64
[root@localhost ~]# rpm -e --nodeps mariadb-libs-5.5.68-1.el7.x86_64
[root@localhost ~]# yum remove mysql-*
```

### 下载rpm文件
``` bash
wget https://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/mysql57-community-release-el7-10.noarch.rpm
```
### 安装Mysql
``` bash
# 安装发行文件
yum -y localinstall mysql57-community-release-el7-10.noarch.rpm
或
rpm -ivh mysql57-community-release-el7-10.noarch.rpm

# 查看mysql版本 
yum whatprovides mysql-community-server

# 原因是Mysql的GPG升级了，需要重新获取
rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022

# 安装mysql 会有报错
yum -y install mysql-community-server 
```
### 启动Mysql
``` bash
systemctl start mysqld
systemctl enable mysqld
```
### 重置密码
``` bash
# 查看临时密码
[root@localhost ~]# cat /var/log/mysqld.log |grep password
2022-08-25T08:25:32.680447Z 1 [Note] A temporary password is generated for root@localhost: XcrEJlRho0_b

# 进入mysql  降低密码要求 可不做
mysql -uroot -p
set global validate_password_policy=0;
set global validate_password_length=1;

set password for root@localhost = password('root');
```
### 开放远程连接
``` bash
use mysql;
update user set user.Host='%' where user.User='root';
flush privileges;
```


## RPM安装
Mysql5.7 RPM[下载地址](https://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/)

### 卸载之前版本

``` bash 
# rpm -e +安装包 或  yum remove +安装包
# rpm -ev --nodeps  +安装包  强制删除

[root@localhost ~]# rpm -qa|grep mariadb
mariadb-libs-5.5.60-1.el7_5.x86_64
[root@localhost ~]# rpm -e --nodeps mariadb-libs-5.5.60-1.el7_5.x86_64
[root@localhost ~]# yum remove mysql-*
```

### 下载rpm文件
mysql-community-libs-compat 能解决缺失libmysqlclient.so.18问题，[处理参考](https://blog.csdn.net/LT_Future/article/details/103648662)
``` bash
wget https://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/mysql-community-common-5.7.31-1.el7.x86_64.rpm
wget https://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/mysql-community-libs-5.7.31-1.el7.x86_64.rpm
wget https://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/mysql-community-client-5.7.31-1.el7.x86_64.rpm
wget https://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/mysql-community-server-5.7.31-1.el7.x86_64.rpm
wget https://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/mysql-community-libs-compat-5.7.31-1.el7.x86_64.rpm
```

### 安装Mysql
``` bash
yum install -y perl numactl

rpm -ivh mysql-community-common-5.7.31-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-5.7.31-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-5.7.31-1.el7.x86_64.rpm
rpm -ivh mysql-community-server-5.7.31-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-compat-5.7.31-1.el7.x86_64.rpm
```

通过在命令行中指定可选的–prefix或–relocate参数来指定安装路径或目录。

（1）指定安装路径：如果我们要将软件包安装到路径/usr/local/newdir/下，我们可以通过执行以下命令来实现：rpm -ivh package.rpm –prefix /usr/local/newdir/。
（2）指定安装目录：如果我们要将RPM软件包中的某一组件安装到路径/usr/lib/newlib/下，我们可以通过执行以下命令来实现：rpm -ivh package.rpm -–relocate ‘/usr/lib=/usr/lib/newlib/’。

在安装RPM时，指定安装路径或目录可以更好地对软件进行定制化安装，提高软件的应用效率。在使用时，我们需要根据具体需要来指定正确的安装目录和路径，同时，为了避免软件安装过程中出现意外情况，我们也需要进行备份等工作。



RPM其他命令-卸载rpm包
卸载包命令：rpm -e yum-utils-1.1.31-54.el7_8
卸载包命令：rpm -e --nodeps yum-utils-1.1.31-54.el7_8
功能：卸载rpm包，-e是erase简写，就是清除卸载包；--nodeps，是代表不确认包的依赖。

RPM其他命令-查看已安装的包
查询已安装的rpm包列表：rpm -qa
已安装包中的查询包含关键字的包：rpm -qa | grep yum-utils

RPM其他命令-查看已安装包的信息
命令：rpm -qi yum-utils

### 启动Mysql
``` bash
systemctl start mysqld
systemctl enable mysqld
```

### 重置密码
``` bash
# 查看临时密码
[root@localhost ~]# cat /var/log/mysqld.log |grep password
2022-08-25T08:25:32.680447Z 1 [Note] A temporary password is generated for root@localhost: IqYx0nr/VEk7

# 进入mysql  降低密码要求 可不做
mysql -uroot -p
set global validate_password_policy=0;
set global validate_password_length=1;

set password for root@localhost = password('root');
```
### 开放远程连接
``` bash
use mysql;
update user set user.Host='%' where user.User='root';
flush privileges;
```
### 开启日志
编辑my.cnf文件，添加如下代码，并重启mysqld
```bash
[root@localhost ~]# vi /etc/my.cnf
# 开启binlog功能，可不加
log-bin=mysql-bin     #binlog文件名
binlog_format=Mixed     #选择row模式
server_id=1           # 为当前服务取一个唯一的 id（MySQL5.7 之后需要配置）

[root@localhost ~]# systemctl restart mysqld
```

## 安装包安装
Mysql5.7安装包[下载地址](https://downloads.mysql.com/archives/community/)

### 卸载之前版本
__卸载mariadb 和 mysql__
``` bash 
# rpm -e +安装包 或  yum remove +安装包
# rpm -ev --nodeps  +安装包  强制删除
# 卸载mariadb 和 mysql
[root@localhost ~]# rpm -qa|grep mariadb
mariadb-libs-5.5.68-1.el7.x86_64
[root@localhost ~]# rpm -e --nodeps mariadb-libs-5.5.68-1.el7.x86_64
[root@localhost ~]# yum remove mysql-*
```

### 创建MySQL用户组
``` bash
# 检查mysql 用户组是否存在
cat /etc/group | grep mysql
cat /etc/passwd |grep mysql

# 创建mysql 用户组和用户
groupadd mysql
useradd -r -g mysql mysql
```

### 获取安装包
``` bash 
wget https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.31-linux-glibc2.12-x86_64.tar.gz
```
### 解压mysql
``` bash
tar -zxvf mysql-5.7.34-linux-glibc2.12-x86_64.tar.gz
mv mysql-5.7.34-linux-glibc2.12-x86_64 /usr/local/mysql/
```

### 创建数据和⽇志⽬录
``` bash
chown -R mysql:mysql /usr/local/mysql
chmod -R 755 /usr/local/mysql
```

### 初始化mysqld
``` bash
#务必记住数据库管理员临时密码  可指定文件夹
/usr/local/mysql/bin/mysqld --initialize --user=mysql --datadir=/usr/local/mysql/data --basedir=/usr/local/mysql
```
![](https://qiufuqi.github.io/img/hexo/20231205135440.png)

### 编写配置文件my.cnf
``` bash
[root@localhost ~]# vi /etc/my.cnf 

[mysqld]
datadir=/usr/local/mysql/data
port = 3306
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
symbolic-links=0
max_connections=400
innodb_file_per_table=1
# 表名大小写不明感
lower_case_table_names=1


#开启binlog功能，可不加
log-bin=mysql-bin     #binlog文件名
binlog_format=Mixed     #选择row模式
server_id=1           # 为当前服务取一个唯一的 id（MySQL5.7 之后需要配置）

binlog 有三种格式：
Statement（Statement-Based Replication,SBR）：每一条会修改数据的 SQL 都会记录在 binlog 中。
Row（Row-Based Replication,RBR）：不记录 SQL 语句上下文信息，仅保存哪条记录被修改。
Mixed（Mixed-Based Replication,MBR）：Statement 和 Row 的混合体。
```
### 启动mysql
``` bash 
# 添加软连接
ln -s /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
ln -s /usr/local/mysql/bin/mysql /usr/bin/mysql

# 重启mysql服务
systemctl daemon-reload
systemctl start mysqld.service
```
### 登录mysql 
``` bash
# 如果安装目录不是/usr/local/mysql 而是指定目录，比如/home/mysql 则需要做一个软连接，否则报错
ln -s /home/mysql /usr/local/mysql

#密码就是初始化时生成的临时密码
mysql -u root -p
```

### 修改密码
``` bash
set password for root@localhost = password('root');
```
### 开放远程连接
``` bash
use mysql;
update user set user.Host='%' where user.User='root';
flush privileges;
```

### 设置开机自启
#### 第一种启动
``` bash
#赋予可执行权限
chmod +x /etc/init.d/mysqld
#添加服务
chkconfig --add mysqld
#显示服务列表
chkconfig --list
```
#### 第二种启动
``` bash
# 如果不能自动启动， 可以编写脚本启动
[root@sx-db init.d]# vi /etc/init.d/mysql.sh
#!/bin/bash
echo "----启动mysql----"$(date +%y年%m月%d日%H:%M:%s);
export LANG=zh-cn.UTF8

systemctl daemon-reload
systemctl start mysqld.service

[root@sx-db init.d]# chmod +x mysql.sh 
# 在rc.local最后添加
[root@sx-db init.d]# chmod +x /etc/rc.d/rc.local
[root@sx-db init.d]# vi /etc/rc.d/rc.local
/etc/init.d/mysql.sh
```
#### 第三种启动
使用systemd，自己写个服务