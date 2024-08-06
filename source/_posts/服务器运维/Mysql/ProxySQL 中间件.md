---
title: ProxySQL中间件
date: 2022-08-27
tags:
  - Linux
  - Mysql
  - ProxySQL
categories: 
- 运维
- 数据库
- ProxySQL
keywords: 'Linux,Mysql,ProxySQL'
description: ProxySQL中间件
cover: https://qiufuqi.github.io/img/hexo/20231205135255.png
abbrlink: ProxySQL
comments: false
---

Mysql中间件--ProxySQL

# ProxySQL简介
ProxySQL是基于 MySQL 的一款开源的中间件的产品，是一个灵活强大的MySQL代理层, 是一个能实实在在用在生产环境的MySQL中间件，可以实现读写分离，支持 Query 路由功能，支持动态指定某个 SQL 进行 cache，支持动态加载配置、故障切换和一些 SQL的过滤功能。
[介绍](https://www.proxysql.com/) [git地址](https://github.com/sysown/proxysql/wiki)

# ProxySQL运行机制
runtime：运行中使用的配置文件
memory：提供用户动态修改配置文件
disk：将修改的配置保存到磁盘SQLit表中（即：proxysql.db）
config：一般不使用它（即：proxysql.cnf）
``` bash
一般，修改的配置都是在memory层。可以load到runtime，使配置在不用重启proxysql的情况下也可以生效，也可以save到disk，将对配置的修改持久化！
+-------------------------+
|         RUNTIME         |
+-------------------------+
      /|\          |
       |           |
  [1]  |       [2] |
       |          \|/
+-------------------------+
|         MEMORY          |
+-------------------------+ 
      /|\          |      |\
       |           |        \
   [3] |       [4] |         \ [5]
       |          \|/         \
+-------------------------+  +---------------+
|          DISK           |  |  CONFIG FILE  |
+-------------------------+  +---------------+
RUNTIME层 代表的是ProxySQL当前生效的配置，包括 global_variables, mysql_servers, mysql_users, mysql_query_rules。无法直接修改这里的配置，必须要从下一层load进来。
MEMORY层 是平时在mysql命令行修改的 main 里头配置，可以认为是SQLite数据库在内存的镜像。我们通过Set修改配置也是先修改此层的内容。
DISK层 持久存储的那份配置，一般在$(DATADIR)/proxysql.db，在重启的时候会从硬盘里加载。 /etc/proxysql.cnf文件只在第一次初始化的时候用到，完了后，如果要修改监听端口等，还是需要在管理命令行里修改，再save到硬盘。

# 修改配置往上层是 LOAD，往下层是 SAVE
#以下命令可用于加载或保存 users (mysql_users):
[1]: LOAD MYSQL USERS TO RUNTIME / LOAD MYSQL USERS FROM MEMORY   #常用。将修改后的配置(在memory层)用到实际生产
[2]: SAVE MYSQL USERS TO MEMORY / SAVE MYSQL USERS FROM RUNTIME        #将生产配置拉一份到memory中
[3]: LOAD MYSQL USERS TO MEMORY / LOAD MYSQL USERS FROM DISK           #将磁盘中持久化的配置拉一份到memory中来
[4]: SAVE MYSQL USERS TO DISK /  SAVE MYSQL USERS FROM MEMORY     #常用。将memoery中的配置保存到磁盘中去
[5]: LOAD MYSQL USERS FROM CONFIG                                      #将配置文件中的配置加载到memeory中

# 以下命令加载或保存servers (mysql_servers):
[1]: LOAD MYSQL SERVERS TO RUNTIME  #常用，让修改的配置生效
[2]: SAVE MYSQL SERVERS TO MEMORY
[3]: LOAD MYSQL SERVERS TO MEMORY
[4]: SAVE MYSQL SERVERS TO DISK     #常用，将修改的配置持久化
[5]: LOAD MYSQL SERVERS FROM CONFIG

# 以下命令加载或保存query rules (mysql_query_rules):
[1]: load mysql query rules to run    #常用
[2]: save mysql query rules to mem
[3]: load mysql query rules to mem
[4]: save mysql query rules to disk   #常用
[5]: load mysql query rules from config

# 以下命令加载或保存 mysql variables (global_variables):
[1]: load mysql variables to runtime
[2]: save mysql variables to memory
[3]: load mysql variables to memory
[4]: save mysql variables to disk
[5]: load mysql variables from config

# 以下命令加载或保存admin variables (select * from global_variables where variable_name like 'admin-%'):
[1]: load admin variables to runtime
[2]: save admin variables to memory
[3]: load admin variables to memory
[4]: save admin variables to disk
[5]: load admin variables from config

```

# ProxySQL启动加载配置
当proxysql启动时，首先读取配置文件CONFIG FILE(/etc/proxysql.cnf)，然后从该配置文件中获取datadir，datadir中配置的是sqlite的数据目录。如果该目录存在，且sqlite数据文件存在，那么正常启动，将sqlite中的配置项读进内存，并且加载进RUNTIME，用于初始化proxysql的运行。如果datadir目录下没有sqlite的数据文件，proxysql就会使用config file中的配置来初始化proxysql，并且将这些配置保存至数据库。sqlite数据文件可以不存在，/etc/proxysql.cnf文件也可以为空，但/etc/proxysql.cnf配置文件必须存在，否则，proxysql无法启动。

# ProxySQL安装
## YUM方式安装
``` bash
# 为了实验方便，关闭所有节点防火墙
[root@localhost ~]# systemctl stop firewalld

# 关闭selinux
[root@localhost ~]# vi /etc/sysconfig/selinux
SELINUX=disabled
[root@localhost ~]# setenforce 0

# 修改名称
[root@localhost ~]# hostnamectl set-hostname proxy-sql
[root@localhost ~]# hostname -f

[root@proxy-sql ~]# vi /etc/yum.repos.d/proxysql.repo
[proxysql_repo]
name= ProxySQL YUM repository
baseurl=http://repo.proxysql.com/ProxySQL/proxysql-1.4.x/centos/\$releasever
gpgcheck=1
gpgkey=http://repo.proxysql.com/ProxySQL/repo_pub_key

[root@proxy-sql ~]# yum clean all
[root@proxy-sql ~]# yum makecache
[root@proxy-sql ~]# yum -y install proxysql

[root@proxy-sql ~]# proxysql --version
ProxySQL version 1.4.16-23-gf954ef3, codename Truls

# 启动ProxySQL   不能用systemctl enable proxysql 非本地应用
[root@proxy-sql ~]# chkconfig proxysql on
[root@proxy-sql ~]# systemctl start proxysql
[root@proxy-sql ~]# systemctl status proxysql

# 启动后会监听两个端口，
# 默认为6032和6033。6032端口是ProxySQL的管理端口，6033是ProxySQL对外提供服务的端口 (即连接到转发后端的真正数据库的转发端口)。
# admin管理接口，默认端口为6032。该端口用于查看、配置ProxySQL
# 接收SQL语句的接口，默认端口为6033，这个接口类似于MySQL的3306端口
[root@proxy-sql ~]# netstat -tunlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:6032            0.0.0.0:*               LISTEN      4859/proxysql       
tcp        0      0 0.0.0.0:6033            0.0.0.0:*               LISTEN      4859/proxysql
```
## RPM包方式安装
``` bash
[root@proxy-sql ~]# wget https://github.com/sysown/proxysql/releases/download/v1.4.8/proxysql-1.4.8-1-centos7.x86_64.rpm
[root@mysql-proxy ~]# rpm -ivh proxysql-1.4.8-1-centos7.x86_64.rpm --force
[root@mysql-proxy ~]# /etc/init.d/proxysql start

# 端口6033是启动了四个线程
[root@mysql-proxy ~]# ss -lntup|grep proxy
tcp    LISTEN     0      128       *:6032                  *:*                   users:(("proxysql",pid=4859,fd=23))
tcp    LISTEN     0      128       *:6033                  *:*                   users:(("proxysql",pid=4859,fd=22))
tcp    LISTEN     0      128       *:6033                  *:*                   users:(("proxysql",pid=4859,fd=21))
tcp    LISTEN     0      128       *:6033                  *:*                   users:(("proxysql",pid=4859,fd=20))
tcp    LISTEN     0      128       *:6033                  *:*                   users:(("proxysql",pid=4859,fd=19))

```
# ProxySQL配置
ProxySQL配置
ProxySQL有配置文件/etc/proxysql.cnf和配置数据库文件/var/lib/proxysql/proxysql.db。
这里需要特别注意：如果存在如果存在"proxysql.db"文件(在/var/lib/proxysql目录下)，则ProxySQL服务只有在第一次启动时才会去读取proxysql.cnf文件并解析；后面启动会就不会读取proxysql.cnf文件了！
如果想要让proxysql.cnf文件里的配置在重启proxysql服务后生效(即想要让proxysql重启时读取并解析proxysql.cnf配置文件)，则需要先删除/var/lib/proxysql/proxysql.db数据库文件，然后再重启proxysql服务。这样就相当于初始化启动proxysql服务了，会再次生产一个纯净的proxysql.db数据库文件(如果之前配置了proxysql相关路由规则等，则就会被抹掉)。
``` bash
[root@proxy-sql ~]# egrep -v "^#|^$" /etc/proxysql.cnf
datadir="/var/lib/proxysql"             #数据目录
admin_variables=
{
	admin_credentials="admin:admin"       #连接管理端的用户名与密码
	mysql_ifaces="0.0.0.0:6032"           #管理端口，用来连接proxysql的管理数据库  不限定ip
}
mysql_variables=
{
	threads=4                              #指定转发端口开启的线程数量
	max_connections=2048
	default_query_delay=0
	default_query_timeout=36000000
	have_compress=true
	poll_timeout=2000
	interfaces="0.0.0.0:6033"              #指定转发端口，用于连接后端mysql数据库的，相当于代理作用
	default_schema="information_schema"
	stacksize=1048576
	server_version="5.5.30"                #指定后端mysql的版本
	connect_timeout_server=3000
	monitor_username="monitor"
	monitor_password="monitor"
	monitor_history=600000
	monitor_connect_interval=60000
	monitor_ping_interval=10000
	monitor_read_only_interval=1500
	monitor_read_only_timeout=500
	ping_interval_server_msec=120000
	ping_timeout_server=500
	commands_stats=true
	sessions_sort=true
	connect_retries_on_failure=10
}
mysql_servers =
(
)
mysql_users:
(
)
mysql_query_rules:
(
)
scheduler=
(
)
mysql_replication_hostgroups=
(
)

# proxysql的数据目录
[root@proxy-sql ~]# ll /var/lib/proxysql/
total 1768
-rw-------. 1 root root  122880 Aug 27 16:02 proxysql.db
-rw-------. 1 root root    1916 Aug 27 16:02 proxysql.log
-rw-r--r--. 1 root root       5 Aug 27 16:02 proxysql.pid
-rw-------. 1 root root 1679360 Sep  6 16:30 proxysql_stats.db
```

# ProxySQL Admin管理接口
当Proxysql启动后，将监听两个端口：
admin管理接口，默认端口为6032。该端口用于查看、配置ProxySQL
接收SQL语句的接口，默认端口为6033，这个接口类似于MySQL的3306端口
![](https://qiufuqi.github.io/img/hexo/20231205133859.png)

提前在服务器上安装Mysql [Mysql安装参考](/centos_mysql5.7)
本地使用admin管理ProxySQL(admin默认管理用户，只允许本地登录)
```bash
[root@proxy-sql ~]# mysql -uadmin -padmin -h127.0.0.1 -P6032

mysql> show databases;
+-----+---------------+-------------------------------------+
| seq | name          | file                                |
+-----+---------------+-------------------------------------+
| 0   | main          |                                     |
| 2   | disk          | /var/lib/proxysql/proxysql.db       |
| 3   | stats         |                                     |
| 4   | monitor       |                                     |
| 5   | stats_history | /var/lib/proxysql/proxysql_stats.db |
+-----+---------------+-------------------------------------+
5 rows in set (0.00 sec)

-  main 内存配置数据库，表里存放后端db实例、用户验证、路由规则等信息。表名以 runtime_开头的表示proxysql当前运行的配置内容，
不能通过dml语句修改，只能修改对应的不以 runtime_ 开头的（在内存）里的表，然后 LOAD 使其生效， SAVE 使其存到硬盘以供下次重启加载。
-  disk 是持久化到硬盘的配置，sqlite数据文件。
-  stats 是proxysql运行抓取的统计信息，包括到后端各命令的执行次数、流量、processlist、查询种类汇总/执行时间等等。
-  monitor 库存储 monitor 模块收集的信息，主要是对后端db的健康/延迟检查。

```
由于 ProxySQL 的配置全部保存在几个自带的库中，所以通过管理接口，可以非常方便地通过发送一些SQL命令去修改 ProxySQL 的配置。 ProxySQL 会解析通过该接口发送的某些对ProxySQL 有效的特定命令，并将其合理转换后发送给内嵌的 SQLite3 数据库引擎去运行

ProxySQL 的配置几乎都是通过管理接口来操作的，通过 Admin 管理接口，可以在线修改几乎所有的配置并使其生效。只有两个变量的配置是必须重启 ProxySQL 才能生效的，它们是：
mysql-threads 和 mysql-stacksize
## 更改ProxySQL的ip和port
``` bash
mysql> select @@admin-mysql_ifaces;
+----------------------+
| @@admin-mysql_ifaces |
+----------------------+
| 0.0.0.0:6032         |
+----------------------+
1 row in set (0.01 sec)

## 修改默认管理端口
mysql> set admin-mysql_ifaces='0.0.0.0:1995';
Query OK, 1 row affected (0.001 sec)

# 载入到runtime层，立即生效；保存到disk层，下午重启服务也能生效。
mysql> load admin variables to runtime;
Query OK, 0 rows affected (0.002 sec)

mysql> save admin variables to disk;
Query OK, 35 rows affected (0.004 sec)

# 查看端口随之改变
[root@localhost proxysql]# ss -antl

```
如果ip配置为0.0.0.0表示不限制ip，但是出于安全考虑，admin用户无论怎么设置都只能在本机登录!!!
## ProxySQL新增管理用户
admin-admin_credentials 变量控制的是admin管理接口的管理员账户。
``` bash
# 查询用户
mysql> select @@admin-admin_credentials;
+---------------------------+
| @@admin-admin_credentials |
+---------------------------+
| admin:admin               |
+---------------------------+
1 row in set (0.00 sec)

# 新增用户
mysql> set admin-admin_credentials="admin:admin;testuser:123456";
Query OK, 1 row affected (0.00 sec)

# 或者
mysql> update global_variables set variable_value = 'admin:admin;testuser:123456' where variable_name = 'admin-admin_credentials';
Query OK, 1 row affected (0.002 sec)


mysql> select @@admin-admin_credentials;
+-----------------------------+
| @@admin-admin_credentials   |
+-----------------------------+
| admin:admin;testuser:123456 |
+-----------------------------+
1 row in set (0.00 sec)

# 载入到runtime层，立即生效；保存到disk层，下午重启服务也能生效。
mysql> load admin variables to runtime;
Query OK, 0 rows affected (0.01 sec)

mysql> save admin variables to disk;
Query OK, 31 rows affected (0.09 sec)

# 验证登录 成功登录
[root@proxy-sql ~]# mysql -utestuser -p123456 -h127.0.0.1 -P6032
```
## ProxySQL新增普通用户
admin-stats_credentials变量控制admin管理接口的普通用户，这个变量中的用户没有超级管理员权限，只能查看monitor库和main库中关于统计的数据，其它库都是不可见的，且没有任何写权限
``` bash

mysql> select @@admin-state_credentials;
Empty set (0.00 sec)

# 添加新的普通用户
mysql> set admin-stats_credentials='statsuser:123456';
Query OK, 1 row affected (0.00 sec)

# 或者
mysql> update global_variables set variable_value = 'statsuser:123456' where variable_name = 'admin-stats_credentials';
Query OK, 1 row affected (0.002 sec)


# 刷新配置文件
mysql> load admin variables to runtime;
Query OK, 0 rows affected (0.01 sec)

mysql> save admin variables to disk;
Query OK, 31 rows affected (0.07 sec)

[root@proxy-sql ~]# mysql -ustatsuser -p123456 -h127.0.0.1 -P6032
mysql> show databases;
+-----+---------------+-------------------------------------+
| seq | name          | file                                |
+-----+---------------+-------------------------------------+
| 0   | main          |                                     |
| 2   | monitor       |                                     |
| 3   | stats_history | /var/lib/proxysql/proxysql_stats.db |
+-----+---------------+-------------------------------------+
3 rows in set (0.00 sec)
```
