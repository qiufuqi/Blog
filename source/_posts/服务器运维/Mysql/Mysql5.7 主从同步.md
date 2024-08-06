---
title: Mysql5.7 主从同步
date: 2022-09-7
tags:
  - Linux
  - Mysql
  - 主从
categories: 
- 运维
- 数据库
- Mysql
- 主从同步
keywords: 'Linux,Mysql,主从'
description: Mysql5.7 主从同步
cover: https://qiufuqi.github.io/img/hexo/20231205134139.png
abbrlink: mysql5.7_MS
comments: false
---

系统都是CentOS7.6，MySQL版本是5.7，准备一主两从架构(基于GTID的同步,两个从库都要开启read_only=on)。
![](https://qiufuqi.github.io/img/hexo/20231205134139.png)
## 环境准备
``` bash
# 三台服务器
10.128.1.51   mysql-master   mysql5.7
10.128.1.52   mysql-slave1   mysql5.7
10.128.1.53   mysql-slave2   mysql5.7

# 为了实验方便，关闭所有节点防火墙
[root@localhost ~]# systemctl stop firewalld

# 关闭selinux
[root@localhost ~]# vi /etc/sysconfig/selinux
SELINUX=disabled

[root@localhost ~]# setenforce 0

# 修改各个节点名称
[root@localhost ~]# hostnamectl set-hostname mysql-master
[root@localhost ~]# hostname -f

# 特别要注意一个关键点: 必须保证各个mysql节点的主机名不一致，并且能通过主机名找到各成员！
[root@localhost ~]# vi /etc/hosts
10.128.1.51   mysql-master
10.128.1.52   mysql-slave1
10.128.1.53   mysql-slave2
```

## 节点MySQL5.7安装
MySQL5.7安装请参考：[安装步骤](/centos_mysql5.7)


## 配置Mysql主从同步
基于GTID的主从同步，在mysql-master 和 mysql-slave1、mysql-slave2节点上操作。
### 主库操作
``` bash
# 1) 主数据mysql-master的操作配置
[root@mysql-master ~]# >/etc/my.cnf
[root@mysql-master ~]# vi /etc/my.cnf
[mysqld]
datadir = /var/lib/mysql
socket = /var/lib/mysql/mysql.sock
        
symbolic-links = 0
        
log-error = /var/log/mysqld.log
pid-file = /var/run/mysqld/mysqld.pid
    
#GTID:
server_id = 51
gtid_mode = on
enforce_gtid_consistency = on
      
#binlog
log_bin = master-bin
log-slave-updates = 1
binlog_format = row
sync-master-info = 1    
sync_binlog = 1        
     
#relay log
skip_slave_start = 1

# 配置完后重启mysql
[root@mysql-master ~]# systemctl restart mysqld

# 主库执行 从库复制授权  可将ip更换为%表示所有从库都可以连，也可以指定从库IP增强安全性
mysql> grant replication slave,replication client on *.* to slave@'10.128.1.52' identified by "Zxc,./123";
Query OK, 0 rows affected, 1 warning (0.04 sec)
mysql> grant replication slave,replication client on *.* to slave@'10.128.1.53' identified by "Zxc,./123";
Query OK, 0 rows affected, 1 warning (0.04 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.02 sec)

# 查询授权信息
mysql> show grants for slave@'10.128.2.52';
+-----------------------------------------------------------------------------+
| Grants for slave@10.128.2.52                                                |
+-----------------------------------------------------------------------------+
| GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave'@'10.128.2.52' |
+-----------------------------------------------------------------------------+
1 row in set (0.00 sec)

# 创建数据库--可省略
mysql> CREATE DATABASE kevin CHARACTER SET utf8 COLLATE utf8_general_ci;
Query OK, 1 row affected (0.04 sec)

mysql> use kevin;
Database changed
mysql> create table if not exists haha (id int(10) PRIMARY KEY AUTO_INCREMENT,name varchar(50) NOT NULL);
Query OK, 0 rows affected (0.20 sec)

mysql> insert into kevin.haha values(1,"congcong"),(2,"huihui"),(3,"grace"); 
Query OK, 3 rows affected (0.07 sec)
Records: 3  Duplicates: 0  Warnings: 0

```

### 从库操作
与主服务器配置大概一致，除了server_id不一致外，从服务器还可以在配置文件里面添加："read_only＝on" ,使从服务器只能进行读取操作，此参数对超级用户无效，并且不会影响从服务器的复制；
``` bash
# 从数据库mysql-slave1 (10.128.1.52，10.128.1.53)的配置操作

# 拷贝的服务器所以需要使server-uuid不一致
[root@mysql-slave1 ~]# vi /var/lib/mysql/auto.cnf

[root@mysql-slave1 ~]# >/etc/my.cnf
[root@mysql-slave1 ~]# vi /etc/my.cnf

[mysqld]
datadir = /var/lib/mysql
socket = /var/lib/mysql/mysql.sock
        
symbolic-links = 0
        
log-error = /var/log/mysqld.log
pid-file = /var/run/mysqld/mysqld.pid
    
#GTID:  id需要修改
server_id = 52
gtid_mode = on
enforce_gtid_consistency = on
      
#binlog
log_bin = master-bin
log-slave-updates = 1
binlog_format = row
sync-master-info = 1
sync_binlog = 1
      
#relay log
skip_slave_start = 1
read_only = on

# 配置完成后重启mysql
[root@mysql-slave1 ~]# systemctl restart mysqld

# 登录mysql，做主从同步
[root@mysql-slave1 ~]# mysql -uroot -p

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

# 执行主从同步
mysql> stop slave;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> change master to master_host='10.128.1.51',master_user='slave',master_password='Zxc,./123',master_auto_position=1;
Query OK, 0 rows affected, 2 warnings (0.21 sec)

# 或者 从指定日志的记录开始
mysql> change master to master_host='10.128.1.51',master_user='slave',master_password='Zxc,./123',master_log_file='mysql-bin.000417',master_log_pos=63642574;

mysql> start slave;
Query OK, 0 rows affected (0.03 sec)

mysql> show slave status \G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.128.1.51
                  Master_User: slave
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: master-bin.000001
          Read_Master_Log_Pos: 4930
               Relay_Log_File: mysql-slave2-relay-bin.000002
                Relay_Log_Pos: 5145
        Relay_Master_Log_File: master-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
............
............ 
            Retrieved_Gtid_Set: dfdda2bb-2e59-11ed-abbe-525400891b9e:1-21
            Executed_Gtid_Set: dfdda2bb-2e59-11ed-abbe-525400891b9e:1-21
                Auto_Position: 1
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)

ERROR: 
No query specified


# 删除slave
mysql> stop slave;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> reset slave all;
Query OK, 0 rows affected, 1 warning (0.00 sec)

```
至此，mysql5.7 主从同步完成。

## 日常处理
主库操作
``` bash
# 主库日志清理
show binary logs;
PURGE BINARY LOGS TO  'mysql-bin.033662';
```

从库操作
``` bash
# 同步出现问题时，可跳过不重要的同步。
stop slave;
# 跳过一条错误
set GLOBAL SQL_SLAVE_SKIP_COUNTER=1;
start slave;
show slave status;
```
