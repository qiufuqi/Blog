---
title: CentOS安装Oracle
date: 2022-09-24
tags:
  - Linux
  - CentOS
  - Oracle
categories: 
- 运维
- 数据库
- Oracle
keywords: 'Linux,CentOS,Oracle'
description: CentOS安装Oracle
cover: https://qiufuqi.github.io/img/hexo/20231205141201.png
abbrlink: centos_oracle
comments: false
---

Linux环境中安装Oracle，系统版本centos7.6 gui 桌面版。
# 前期准备
## 资源准备
提前下载好相关资源，并存放/home/soft目录下
链接：https://pan.baidu.com/s/1XniDCJmyYvsYcXihV5Bqjw 
提取码：o0nv
![](https://qiufuqi.github.io/img/hexo/20220923111716.png)

## 服务器准备
``` bash
# 防火墙放行端口（或者关闭防火墙）  关闭selinux  
[root@localhost ~]# firewall-cmd --zone=public --add-port=1521/tcp --permanent
[root@localhost ~]# firewall-cmd --reload

[root@localhost ~]# sed -i 's#SELINUX=.*#SELINUX=disabled#g' /etc/selinux/config
[root@localhost ~]# setenforce 0
```
## oracle用户准备
``` bash
[root@localhost ~]# groupadd oinstall
[root@localhost ~]# groupadd dba
[root@localhost ~]# useradd -g oinstall -G dba oracle
useradd: user 'oracle' already exists
[root@localhost ~]# id oracle
uid=1000(oracle) gid=1000(oracle) groups=1000(oracle),10(wheel)

[root@localhost ~]# mkdir -p /home/data/oracle              #创建oracle安装目录
[root@localhost ~]# mkdir -p /home/data/database            #创建oracle解压目录
[root@localhost ~]# mkdir -p /home/data/oraInventory        #创建oracle配置文件目录~~~~
[root@localhost ~]# chown -R oracle:oinstall /home/data     #设置oracle用户为目录的所有者
[root@localhost ~]# chmod -R 775 /home/data
```

## 修改oracle用户限制
``` bash
# 修改操作系统对oracle用户资源的限制
[root@localhost ~]# vi /etc/security/limits.conf
······
oracle soft nproc 4096
oracle hard nproc 16384
oracle soft nofile 2048
oracle hard nofile 65536
# End of file

# 使limits.conf文件配置生效，必须要确保pam_limits.so文件被加入到启动文件中
[root@localhost ~]# vi /etc/pam.d/login
······
session    required    /lib/security/pam_limits.so
session    required    pam_limits.so

# 设置其最大可启动进程数与最多可开启文件数
[root@localhost ~]# vi /etc/profile
······
if [ $USER = "oracle" ]; then
   if [ $SHELL = "/bin/ksh" ]; then
           ulimit -p 16384
           ulimit -n 65536
   else
           ulimit -u 16384 -n 65536
   fi
fi

# 配置修改生效
[root@localhost ~]# source /etc/profile
```
## 配置内核参数和资源限制
``` bash
# 每次操作系统启动时，便会自动设置这些内核参数
[root@localhost ~]# vi /etc/sysctl.conf
fs.aio-max-nr = 1048576
fs.file-max = 6815744
kernel.shmall = 2097152
kernel.shmmax = 536870912
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576

# 查看并生效
[root@localhost ~]# sysctl -p
```
## 配置oracle用户环境变量
将下列设置添加到 /home/oracle/.bash_profile文件中。
注意:要写到原有“PATH=PATH: PATH: HOME/bin”变量上面，否则会提示“bash: sqlplus: command not found”
``` bash
[root@localhost ~]# vi /home/oracle/.bash_profile
·········
umask 022
export ORACLE_BASE=/home/data/oracle
export ORACLE_HOME=$ORACLE_BASE/product/11.2.0/db_1
export ORACLE_SID=orcl
export ORACLE_TERM=xterm
export PATH=$ORACLE_HOME/bin:/user/sbin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export LANG=C
export NLS_LANG="SIMPLIFIED CHINESE_CHINA.AL32UTF8"
PATH=$PATH:$HOME/.local/bin:$HOME/bin:$ORACLE_HOME/bin
export PATH

# 切换oracle用户，执行source /home/oracle/.bash_profile 使用echo $ORACLE_HOME 来显示是否生效
[root@localhost ~]# su - oracle
[oracle@localhost ~]$ source /home/oracle/.bash_profile
```
## 配置主机名监听
``` bash
[root@oracle ~]# hostnamectl set-hostname oracle
[root@oracle ~]# vi /etc/sysconfig/network
·········
hostname=oracle
# 本机ip
[root@oracle ~]# vi /etc/hosts
·········
10.128.1.71 oracle
```
## 安装java指定jdk
jdk-8u60-linux-x64.tar.gz
``` bash
[root@oracle soft]# vi /home/soft/install_jdk.sh
#!/bin/bash
# sources variables 
check_user=`id -u`
if [ ${check_user} != "0" ];then
		echo "Must be root can use !"
		exit 1
fi
JDK_version="jdk1.8.0_60"
BASE_dir="/usr/java"
SOFT_dir="/home/soft"
JDK_package="jdk-8u60-linux-x64.tar.gz"
JAVA_HOME="/usr/java/${JDK_version}"
source /etc/profile


function jdk_install(){
	 [ -d ${SOFT_dir} ] || mkdir -p ${SOFT_dir}
	 [ -e ${BASE_dir}/${JDK_version} ] && echo -e "\033[32m    ${JDK_version}已部署,请退出!!! \033[0m" 
	if [ -e ${BASE_dir}/${JDK_version} ];then
		 sleep 1 && exit 0
	else
	   [ -f ${SOFT_dir}/${JDK_package} ] || echo -e "\033[36m    ${JDK_package}正在下载.... \033[0m"
	   sleep 1
	   [ -d "${BASE_dir}" ] && rm -rf ${BASE_dir}/* || mkdir -p ${BASE_dir}
	   sleep 1 &&  echo -e "\033[35m    正在解压${JDK_package}.... \033[0m"
	   cd ${SOFT_dir} && tar -xzf  ${JDK_package} -C ${BASE_dir}
	   sleep 2;
	   sed  -i '/JAVA_HOME*/d' /etc/profile
	   sed -i.ori '$a export JAVA_HOME=/usr/java/jdk1.8.0_60\nexport PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$PATH\nexport CLASSPATH=.$CLASSPATH:$JAVA_HOME/lib:$JAVA_HOME/jre/lib:$JAVA_HOME/lib/tools.jar' /etc/profile
	   source /etc/profile && . /etc/profile
	   echo "JAVA_HOME=$JAVA_HOME"  
	   echo `$JAVA_HOME/bin/java -version`
	   echo -e "\033[32m    JDK已部署成功!!! \033[0m" 
   fi
}
function main(){
			 jdk_install
}
main

[root@oracle soft]# sh /home/soft/install_jdk.sh
[root@oracle soft]# sh install_jdk.sh 
    正在解压jdk-8u60-linux-x64.tar.gz.... 
JAVA_HOME=/usr/java/jdk1.8.0_60
java version "1.8.0_60"
Java(TM) SE Runtime Environment (build 1.8.0_60-b27)
Java HotSpot(TM) 64-Bit Server VM (build 25.60-b23, mixed mode)

    JDK已部署成功!!! 
```
# 开始安装oracle
## 安装相关依赖包
``` bash
[root@oracle soft]# yum -y install binutils-* compat-libcap1-* gcc-* gcc-c++-* glibc-* glibc-devel-* glibc-headers-* libstdc* elfutils-libelf-devel* libaio-devel* unixODBC-* pdksh-* libaio-* libgcc-* libXi-* libXtst-* make-* sysstat-* ld-linux.so.2 libc.so.6*

# 默认pdksh会无法安装，使用rpm安装 
# https://ftp.icm.edu.pl/packages/linux-redhat/linux/6.1/en/os/i386/RedHat/RPMS/
[root@oracle soft]# wget http://vault.centos.org/5.11/os/i386/CentOS/pdksh-5.2.14-37.el5_8.1.i386.rpm
[root@oracle soft]# rpm -ivh pdksh-5.2.14-37.el5_8.1.i386.rpm
```
## 安装图形化界面（已安装可忽略）
``` bash
systemctl get-default
yum grouplist
yum groupinstall "GNOME Desktop" "Graphical Administration Tools"
# 更新系统的运行级别为graphical.target，设置默认启动图形界面:
systemctl set-default graphical.target
ln -sf /lib/systemd/system/graphical.target /etc/systemd/system/default.target
systemctl get-default  #检查一下
reboot
```
## 解压oracle安装包
``` bash
[root@oracle soft]# unzip linux.x64_11gR2_database_1of2.zip -d /home/data/
[root@oracle soft]# unzip linux.x64_11gR2_database_2of2.zip -d /home/data/

# /home/oracle/database 有执行权限，将该目录赋予oracle帐号所有，并拥有执行权限
[root@oracle soft]# chmod -R 700 /home/data/database 
[root@oracle soft]# chown -R oracle:oinstall /home/data/database
```
## 开始安装

**注意此步骤一定要在图形桌面上执行**
``` bash
# root用户执行



[root@oracle ~]# yum whatprovides "*/xhost"
# 根据上条命令安装xorg-x11-server-utils-7.7-20.el7.x86_64
[root@oracle ~]# yum -y install xorg-x11-server-utils-7.7-20.el7.x86_64

# 进入ROOT用户
[root@oracle ~]# su – 
[root@oracle ~]# DISPLAY=:0.0; export DISPLAY
[root@oracle ~]# cd /usr/bin
[root@oracle ~]# ./xhost
[root@oracle ~]# ./xhost +


# 切换为oracle
[root@oracle ~]# su - oracle
[oracle@oracle ~]# su - oracle
[oracle@oracle ~]# DISPLAY=:0.0; export DISPLAY
[oracle@oracle ~]# cd /home/data/database
[oracle@oracle ~]# ./runInstaller  # 出现图形化安装页面
```
![](https://qiufuqi.github.io/img/hexo/20220923154134.png)

## 安装步骤
取消选中这个界面上的I wish to receive security updates via My Oracle Support 复选框，点击Next
![](https://qiufuqi.github.io/img/hexo/20220923154308.png)
直接默认yes，点击下一步，默认创建和配置一个数据库
![](https://qiufuqi.github.io/img/hexo/20220923154329.png)
 选择服务类
![](https://qiufuqi.github.io/img/hexo/20220923154341.png)
选择单实例库
![](https://qiufuqi.github.io/img/hexo/20220923154410.png)
选择典型安装，也可以选择高级安装，安装步骤更多：
![](https://qiufuqi.github.io/img/hexo/20220923154436.png)
安装Oracle基本配置：最好保持和ORACLE_BASE&&ORACLE_HOME配置环境变量一致，点击yes：
密码：Oracle2022  （大小写字母+数字）
![](https://qiufuqi.github.io/img/hexo/20220923154515.png)
选择清单目录、即Oracle配置文件存放目录：用户组选择默认
![](https://qiufuqi.github.io/img/hexo/20220923154804.png)
 先决条件检查：查看缺失的依赖包
![](https://qiufuqi.github.io/img/hexo/20220923154908.png)
![](https://qiufuqi.github.io/img/hexo/20220923154923.png)

最好不要选择右上角“Igrnore all”(全部忽略)，如下图显示，有些包还没有安装，里面显示是需要32位(i386)的，相关文件已经打包好。
使用方法：在root用户下，解压后直接./oracle_rpm_setup.sh即可自动安装全部的包。
``` bash
[root@oracle ~]# cd /home/soft/
[root@oracle ~]# unzip oracle_rpm_setup.zip
[root@oracle ~]# cd oracle_rpm_setup/
[root@oracle ~]# ./oracle_rpm_setup.sh
```
点击"Check Again"后，之前提示包全部完成，剩下的可以忽略。
![](https://qiufuqi.github.io/img/hexo/20220923155103.png)
直接下一步，在Summary界面，保持默认，点击Finish，开始安装：
![](https://qiufuqi.github.io/img/hexo/20220923155132.png)
安装过程中，差不多需要15—30分钟左右，其中会有一些错误提示，不过不影响，我们选择continue和默认即可。
![](https://qiufuqi.github.io/img/hexo/20220923155157.png)
安装完成后会提示需要执行两个脚本， 使用root账户执行两个脚本即可完成所有安装步骤。
![](https://qiufuqi.github.io/img/hexo/20220923155212.png)
登录root用户，到下面的两个目录下执行脚本即可。
``` bash
[root@oracle ~]# cd /home/data/oraInventory/
[root@oracle ~]# sh orainstRoot.sh
[root@oracle ~]# cd /home/data/oracle/product/11.2.0/db_1
[root@oracle ~]# sh root.sh
```
点击close，至此完成Oracle的配置和安装：
![](https://qiufuqi.github.io/img/hexo/20220923155457.png)

## 启动oracle和配置监听
linux下的Oracle在安装结束后是处于运行状态的。端口号1521，运行top -u oracle可以看到以Oracle用户运行的进程。在图形化界面下，运行$ORACLE_HOME/sqldeveloper/sqldeveloper.sh可以出现Oracle自带的免费Oracle管理客户端SQL Developer。试着连接刚安装的Oracle，连接成功。
![](https://qiufuqi.github.io/img/hexo/20220923171348.png)

## 以oracle用户登录
以oracle身份登录数据库，进入Sqlplus控制台：
``` bash
[root@oracle db_1]# su - oracle
[oracle@oracle ~]$ sqlplus / as sysdba

[oracle@oracle ~]$ sqlplus /nolog    --进入Sqlplus控制台
# --以系统管理员登录
SQL> connect / as sysdba
Connected.
SQL> startup    
ORA-01081: 无法启动已在运行的 ORACLE - 请首先关闭它
SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> startup    启动
ORACLE instance started.

Total System Global Area 3273641984 bytes
Fixed Size		    2217792 bytes
Variable Size		 1795164352 bytes
Database Buffers	 1459617792 bytes
Redo Buffers		   16642048 bytes
Database mounted.
Database opened.

```
## 启动监听服务
以oracle身份登录数据库，前提是首先启动数据库，也可以用dbstart和dbshut启动和关闭数据库实例
``` bash
[oracle@oracle ~]$ dbstart $ORACLE_HOME #重启oracle实例
[oracle@oracle ~]$ dbshut $ORACLE_HOME  #关闭oracle实例
[oracle@oracle ~]$ lsnrctl status #查看监听状态
[oracle@oracle ~]$ lsnrctl stop   #关闭监听，1521端口关闭
[oracle@oracle ~]$ lsnrctl start    #启动监听，1521端口开启
[oracle@oracle ~]$ dbca        #创建数据库实例orcl，图形界面操作

[oracle@oracle ~]$ sqlplus / as sysdba
```
![](https://qiufuqi.github.io/img/hexo/20220923174929.png)

## 添加用户授权
``` bash
# 创建用户
create user 用户 identified by 密码;
	
# 授权命令
语法： grant connect, resource to 用户名;
例子： grant connect, resource to test;
# 撤销权限
语法： revoke connect, resource from 用户名;
列子： revoke connect, resource from test;

# 某个表授权  ECO9表的拥有者
grant select on ECO9.FORMTABLE_MAIN_94 to XJZT;
grant select on ECO9.hrmsubcompany  to XJZT;

# NC
grant select on yurun501.bd_accsubj to yanfa519;
```

## 密码过期
``` bash
# 进入oracle用户
$sqlplus / as sysdba
sql> alter user smsc identified by <原来的密码> ----不用换新密码

# --查询Orcal密码的有效期设置，LIMIT字段是密码有效天数。
SELECT * FROM dba_profiles WHERE profile='DEFAULT' AND resource_name='PASSWORD_LIFE_TIME';
# --去除180天的密码生存周期的限制可通过如下SQL语句将其关闭
ALTER PROFILE DEFAULT LIMIT PASSWORD_LIFE_TIME UNLIMITED;
```
## 开机自启动
方法一：
``` bash
# 修改/etc/oratab文件 修改为Y
[root@oracle db_1]# vi /etc/oratab
·········
orcl:/home/data/oracle/product/11.2.0/db_1:Y

# 把lsnrctl start和dbstart添加到rc.local文件中
chmod +x /etc/rc.d/rc.local
vi /etc/rc.d/rc.local
# 第一行为开机启动数据库监听服务，第二行为开机启动数据库。(路径跟安装路径相关)。
su - oracle -lc "/home/data/oracle/product/11.2.0/db_1/bin/lsnrctl start"
su - oracle -lc "/home/data/oracle/product/11.2.0/db_1/bin/dbstart $ORACLE_HOME"

# 开机后查看是否启动 oracle用户
[oracle@oracle ~]$ lsnrctl status LISTENER
```
方法二：
``` bash
[root@oracle ~]# vi /usr/lib/systemd/system/oracle.service

[Unit]
Description=Oracle Database 12c Startup/Shutdown Service
After=syslog.target network.target
[Service]
LimitMEMLOCK=infinity
LimitNOFILE=65535
Type=oneshot
RemainAfterExit=yes
User=oracle
Environment="ORACLE_HOME=/home/data/oracle/product/11.2.0/db_1"
ExecStart=/home/data/oracle/product/11.2.0/db_1/bin/dbstart $ORACLE_HOME >> 2>&1 &
ExecStop=/home/data/oracle/product/11.2.0/db_1/bin/dbshut $ORACLE_HOME 2>&1 &
[Install]
WantedBy=multi-user.target

[root@oracle ~]# systemctl enable oracle
[root@oracle ~]# systemctl start oracle
[root@oracle ~]# systemctl stop oracle
[root@oracle ~]# systemctl status oracle
```

## 问题处理
表空间用完时新增表空间
```bash
select file_id,tablespace_name,bytes/1024/1024/1024,maxbytes/1024/1024/1024,AUTOEXTENSIBLE from dba_data_files where tablespace_name='BWDATA';

SELECT
    TABLESPACE_NAME,
    FILE_NAME,
    BYTES / 1024 / 1024 AS SIZE_MB,
    AUTOEXTENSIBLE,
    MAXBYTES / 1024 / 1024 AS MAX_SIZE_MB
FROM
    DBA_DATA_FILES;

USERS	/u01/app/oracle/oradata/yxdb/users01.dbf	5	YES	32767.984375
UNDOTBS1	/u01/app/oracle/oradata/yxdb/undotbs01.dbf	2950	YES	32767.984375
SYSAUX	/u01/app/oracle/oradata/yxdb/sysaux01.dbf	11970	YES	32767.984375
SYSTEM	/u01/app/oracle/oradata/yxdb/system01.dbf	3048	YES	32767.984375
BWDATA	/u01/app/oracle/oradata/yxdb/bwdata01.dbf	32767.984375	YES	32767.984375
BWDATA	/u01/app/oracle/oradata/yxdb/bwdata02.dbf	32767	YES	32767
BWDATA	/u01/app/oracle/oradata/yxdb/bwdata03.dbf	32767	YES	32767
BWDATA	/u01/app/oracle/oradata/yxdb/bwdata04.dbf	22528	YES	22528

# bwdata04往后顺延，前提是磁盘空间足够
ALTER TABLESPACE bwdata ADD DATAFILE '/u01/app/oracle/oradata/yxdb/bwdata04.dbf' SIZE 22528M;
```











