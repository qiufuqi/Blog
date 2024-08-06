---
title: ProxySQL 主从读写分离
date: 2022-09-07
tags:
  - Linux
  - Mysql
  - ProxySQL
categories: 
- 运维
- 数据库
- ProxySQL
- 主从同步
keywords: 'Linux,Mysql,ProxySQL,主从'
description: ProxySQL 主从读写分离
cover: https://qiufuqi.github.io/img/hexo/20220908102559.png
abbrlink: ProxySQL_MS
comments: false
---

ProxySQL实现主从读写分离
ProxySQL是灵活强大的MySQL代理层, 是一个能实实在在用在生产环境的MySQL中间件，可以实现读写分离，支持 Query 路由功能，支持动态指定某个 SQL 进行 cache，支持动态加载配置、故障切换和一些 SQL的过滤功能。
本实验前期准备：
- [ProxySQL中间件](ProxySQL) 
- [Mysql5.7主从同步](/mysql5.7_MS)
  
![](https://qiufuqi.github.io/img/hexo/20220908102559.png)
**ProxySQL实现主从读写分离** [参考地址](https://www.cnblogs.com/kevingrace/p/10329714.html)

## 环境准备
四台服务器 10.128.1.51，10.128.1.52，10.128.1.53已实现mysql主从同步，[搭建参考](/mysql5.7_MS)
``` bash
# 四台服务器
10.128.1.44   proxy-sql
10.128.1.51   mysql-master   mysql5.7
10.128.1.52   mysql-slave1   mysql5.7
10.128.1.53   mysql-slave2   mysql5.7
```
## 添加MySQL节点
``` bash
# 使用insert语句添加主机到mysql_servers表中，其中：hostgroup_id 为10表示写组，为20表示读组。

[root@proxy-sql ~]# mysql -uadmin -padmin -P6032 -h127.0.0.1
··············
mysql> insert into mysql_servers(hostgroup_id,hostname,port) values(10,'10.128.1.51',3306);
Query OK, 1 row affected (0.00 sec)

mysql> insert into mysql_servers(hostgroup_id,hostname,port) values(10,'10.128.1.52',3306);
Query OK, 1 row affected (0.00 sec)

mysql> insert into mysql_servers(hostgroup_id,hostname,port) values(10,'10.128.1.53',3306);
Query OK, 1 row affected (0.00 sec)

==========================================================================================================
如果在插入过程中，出现报错：
ERROR 1045 (#2800): UNIQUE constraint failed: mysql_servers.hostgroup_id, mysql_servers.hostname, mysql_servers.port

说明可能之前就已经定义了其他配置，可以清空这张表 或者 删除对应host的配置
mysql> select * from mysql_servers;
mysql> delete from mysql_servers;
Query OK, 6 rows affected (0.000 sec)
==========================================================================================================

查看这3个节点是否插入成功，以及它们的状态。
mysql> select * from mysql_servers\G;
*************************** 1. row ***************************
       hostgroup_id: 10
           hostname: 10.128.1.51
               port: 3306
             status: ONLINE
             weight: 1
        compression: 0
    max_connections: 1000
max_replication_lag: 0
            use_ssl: 0
     max_latency_ms: 0
            comment: 
*************************** 2. row ***************************
       hostgroup_id: 10
           hostname: 10.128.1.52
               port: 3306
             status: ONLINE
             weight: 1
        compression: 0
    max_connections: 1000
max_replication_lag: 0
            use_ssl: 0
     max_latency_ms: 0
            comment: 
*************************** 3. row ***************************
       hostgroup_id: 10
           hostname: 10.128.1.53
               port: 3306
             status: ONLINE
             weight: 1
        compression: 0
    max_connections: 1000
max_replication_lag: 0
            use_ssl: 0
     max_latency_ms: 0
            comment: 
3 rows in set (0.01 sec)

# 修改后，加载到RUNTIME，并保存到disk
mysql> load mysql servers to runtime;
Query OK, 0 rows affected (0.01 sec)

mysql> save mysql servers to disk;
Query OK, 0 rows affected (0.29 sec)

```
## 监控后端MySQL节点
添加Mysql节点之后，还需要监控这些后端节点。对于后端是主从复制的环境来说，这是必须的，因为ProxySQL需要通过每个节点的read_only值来自动调整它们是属于读组还是写组。
首先在后端master主数据节点上创建一个用于监控的用户名(只需在master上创建即可，因为会复制到slave上)，这个用户名只需具有USAGE权限即可。如果还需要监控复制结构中slave是否严重延迟于master(这个俗语叫做"拖后腿"，术语叫做"replication lag")，则还需具备replication client权限。
``` bash
# 在mysql-master主数据库节点行执行： 10.128.1.51
[root@mysql-master ~]# mysql -uroot -p
··············

mysql> create user monitor@'10.128.1.%' identified by 'Zxc,./123';
Query OK, 0 rows affected (0.05 sec)

mysql> grant replication client on *.* to monitor@'10.128.1.%';
Query OK, 0 rows affected (0.04 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.06 sec)


# mysql-proxy代理层节点上配置监控 10.128.1.44
[root@proxy-sql ~]# mysql -uadmin -padmin -P6032 -h127.0.0.1

mysql> set mysql-monitor_username='monitor';
Query OK, 1 row affected (0.00 sec)

mysql> set mysql-monitor_password='Zxc,./123';
Query OK, 1 row affected (0.00 sec)

# 修改后，加载到RUNTIME，并保存到disk
mysql> load mysql variables to runtime;
Query OK, 0 rows affected (0.00 sec)

mysql> save mysql variables to disk;
Query OK, 98 rows affected (0.08 sec)


# 验证监控结果：ProxySQL监控模块的指标都保存在monitor库的log表中。

# 以下是连接是否正常的监控(对connect指标的监控)：
# 注意：可能会有很多connect_error，这是因为没有配置监控信息时的错误，配置后如果connect_error的结果为NULL则表示正常。
mysql> select * from mysql_server_connect_log;
+-------------+------+------------------+-------------------------+----------------------------------------------------------------------+
| hostname    | port | time_start_us    | connect_success_time_us | connect_error                                                        |
+-------------+------+------------------+-------------------------+----------------------------------------------------------------------+
| 10.128.1.51 | 3306 | 1662607613523034 | 0                       | Access denied for user 'monitor'@'10.128.1.44' (using password: YES) |
| 10.128.1.52 | 3306 | 1662607672716630 | 0                       | Access denied for user 'monitor'@'10.128.1.44' (using password: YES) |
| 10.128.1.53 | 3306 | 1662607673475788 | 0                       | Access denied for user 'monitor'@'10.128.1.44' (using password: YES) |
| 10.128.1.52 | 3306 | 1662607710748387 | 5273                    | NULL                                                                 |
| 10.128.1.53 | 3306 | 1662607711409549 | 4905                    | NULL                                                                 |
| 10.128.1.51 | 3306 | 1662607712070635 | 5543                    | NULL                                                                 |
+-------------+------+------------------+-------------------------+----------------------------------------------------------------------+

# 以下是对心跳信息的监控(对ping指标的监控)
mysql> select * from mysql_server_ping_log;
+-------------+------+------------------+----------------------+----------------------------------------------------------------------+
| hostname    | port | time_start_us    | ping_success_time_us | ping_error                                                           |
+-------------+------+------------------+----------------------+----------------------------------------------------------------------+
| 10.128.1.51 | 3306 | 1662607707491171 | 0                    | Access denied for user 'monitor'@'10.128.1.44' (using password: YES) |
| 10.128.1.53 | 3306 | 1662607707595621 | 0                    | Access denied for user 'monitor'@'10.128.1.44' (using password: YES) |
| 10.128.1.52 | 3306 | 1662607707700053 | 0                    | Access denied for user 'monitor'@'10.128.1.44' (using password: YES) |
| 10.128.1.53 | 3306 | 1662607710806583 | 1621                 | NULL                                                                 |
| 10.128.1.51 | 3306 | 1662607710889190 | 1627                 | NULL                                                                 |
| 10.128.1.52 | 3306 | 1662607710971798 | 1569                 | NULL                                                                 |
+-------------+------+------------------+----------------------+----------------------------------------------------------------------+

# read_only日志此时也为空(正常来说，新环境配置时，这个只读日志是为空的)
mysql> select * from mysql_server_read_only_log;
Empty set (0.00 sec)

# replication_lag的监控日志为空
mysql> select * from mysql_server_replication_lag_log;
Empty set (0.00 sec)

# 指定写组的id为10，读组的id为20
mysql> insert into mysql_replication_hostgroups values(10,20,1);
Query OK, 1 row affected (0.00 sec)

# 在该配置加载到RUNTIME生效之前，先查看下各mysql server所在的组。
mysql> select hostgroup_id,hostname,port,status,weight from mysql_servers;
+--------------+-------------+------+--------+--------+
| hostgroup_id | hostname    | port | status | weight |
+--------------+-------------+------+--------+--------+
| 10           | 10.128.1.51 | 3306 | ONLINE | 1      |
| 10           | 10.128.1.52 | 3306 | ONLINE | 1      |
| 10           | 10.128.1.53 | 3306 | ONLINE | 1      |
+--------------+-------------+------+--------+--------+
3 rows in set (0.00 sec)

# 3个节点都在hostgroup_id=10的组中。
# 现在，将刚才mysql_replication_hostgroups表的修改加载到RUNTIME生效。

mysql> load mysql servers to runtime;
Query OK, 0 rows affected (0.01 sec)

mysql> save mysql servers to disk;
Query OK, 0 rows affected (0.29 sec)


一加载，Monitor模块就会开始监控后端的read_only值，当监控到read_only值后，就会按照read_only的值将某些节点自动移动到读/写组。

例如，此处所有节点都在id=10的写组，slave1和slave2都是slave，它们的read_only=1，这两个节点将会移动到id=20的组。
如果一开始这3节点都在id=20的读组，那么移动的将是Master节点，会移动到id=10的写组。

# 现在看结果
mysql> select hostgroup_id,hostname,port,status,weight from mysql_servers;
+--------------+-------------+------+--------+--------+
| hostgroup_id | hostname    | port | status | weight |
+--------------+-------------+------+--------+--------+
| 10           | 10.128.1.51 | 3306 | ONLINE | 1      |
| 20           | 10.128.1.53 | 3306 | ONLINE | 1      |
| 20           | 10.128.1.52 | 3306 | ONLINE | 1      |
+--------------+-------------+------+--------+--------+
3 rows in set (0.00 sec)

mysql> select * from mysql_server_read_only_log;
+-------------+------+------------------+-----------------+-----------+-------+
| hostname    | port | time_start_us    | success_time_us | read_only | error |
+-------------+------+------------------+-----------------+-----------+-------+
| 10.128.1.53 | 3306 | 1662608321363430 | 2840            | 1         | NULL  |
| 10.128.1.52 | 3306 | 1662608321382341 | 2904            | 1         | NULL  |
| 10.128.1.51 | 3306 | 1662608321401253 | 2136            | 0         | NULL  |
+-------------+------+------------------+-----------------+-----------+-------+
```
## 配置mysql_users
**本实验：未配置路由表的情况下，用户root和sqlsender都落在了group 10 组，全部拥有写入权限，可将sqlsender落在group 20组，只有用读权限，实现了不同用户读写分离。**

上面的所有配置都是关于后端MySQL节点的，现在可以配置关于SQL语句的，包括：发送SQL语句的用户、SQL语句的路由规则、SQL查询的缓存、SQL语句的重写等等。本小节是SQL请求所使用的用户配置，例如root用户。这要求我们需要先在后端MySQL节点添加好相关用户。这里以root和sqlsender两个用户名为例.
``` bash
# 在mysql-master主数据库节点行执行： 10.128.1.51
[root@mysql-master ~]# mysql -uroot -p
········

mysql> grant all on *.* to root@'10.128.1.%' identified by 'Zxc,./123';
Query OK, 0 rows affected, 1 warning (0.04 sec)

mysql> grant all on *.* to sqlsender@'10.128.1.%' identified by 'Zxc,./123';
Query OK, 0 rows affected, 1 warning (0.04 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.04 sec)


# mysql-proxy代理层节点，配置mysql_users表，将刚才的两个用户添加到该表中。 10.128.1.44
[root@proxy-sql ~]# mysql -uadmin -padmin -P6032 -h127.0.0.1
·········

mysql> insert into mysql_users(username,password,default_hostgroup) values('root','Zxc,./123',10);
Query OK, 1 row affected (0.00 sec)

mysql> insert into mysql_users(username,password,default_hostgroup) values('sqlsender','Zxc,./123',10);
Query OK, 1 row affected (0.00 sec)

# 修改后，加载到RUNTIME，并保存到disk
mysql> load mysql users to runtime;
Query OK, 0 rows affected (0.00 sec)

mysql> save mysql users to disk;
Query OK, 0 rows affected (0.14 sec)

```

mysql_users表有不少字段，最主要的三个字段为username、password和default_hostgroup：
-  username：前端连接ProxySQL，以及ProxySQL将SQL语句路由给MySQL所使用的用户名。
-  password：用户名对应的密码。可以是明文密码，也可以是hash密码。如果想使用hash密码，可以先在某个MySQL节点上执行
   select password(PASSWORD)，然后将加密结果复制到该字段。
-  default_hostgroup：该用户名默认的路由目标。例如，指定root用户的该字段值为10时，则使用root用户发送的SQL语句默认
   情况下将路由到hostgroup_id=10组中的某个节点。

``` bash
# 查询用户  只有active=1的用户才是有效的用户。
transaction_persistent字段，当它的值为1时，表示事务持久化：当某连接使用该用户开启了一个事务后，
那么在事务提交/回滚之前，所有的语句都路由到同一个组中，避免语句分散到不同组。
在以前的版本中，默认值为0，不知道从哪个版本开始，它的默认值为1。
我们期望的值为1，所以在继续下面的步骤之前，先查看下这个值，如果为0，则执行下面的语句修改为1。

mysql> select * from mysql_users \G;
*************************** 1. row ***************************
              username: root
              password: Zxc,./123
                active: 1
               use_ssl: 0
     default_hostgroup: 10
        default_schema: NULL
         schema_locked: 0
transaction_persistent: 1
          fast_forward: 0
               backend: 1
              frontend: 1
       max_connections: 10000
*************************** 2. row ***************************
              username: sqlsender
              password: Zxc,./123
                active: 1
               use_ssl: 0
     default_hostgroup: 10
        default_schema: NULL
         schema_locked: 0
transaction_persistent: 1
          fast_forward: 0
               backend: 1
              frontend: 1
       max_connections: 10000
2 rows in set (0.00 sec)

ERROR: 
No query specified

# 当transaction_persistent不为1时，可修改对应用户。
mysql> update mysql_users set transaction_persistent=1 where username='root';
Query OK, 1 row affected (0.000 sec)
 
mysql> update mysql_users set transaction_persistent=1 where username='sqlsender';
Query OK, 1 row affected (0.000 sec)
 
mysql> load mysql users to runtime;
Query OK, 0 rows affected (0.001 sec)
 
mysql> save mysql users to disk;
Query OK, 0 rows affected (0.123 sec)

```
使用roothe和sqlsender用户测试是否能路由到默认的hostgroup_id=10，读写数据。
通过转发端口6033链接，链接是真正的数据库
``` bash
# 测试root用户
[root@proxy-sql ~]# mysql -uroot -p'Zxc,./123' -P6033 -h127.0.0.1 -e "select @@server_id"
mysql: [Warning] Using a password on the command line interface can be insecure.
+-------------+
| @@server_id |
+-------------+
|          51 |
+-------------+
[root@proxy-sql ~]# mysql -uroot -p'Zxc,./123' -P6033 -h127.0.0.1 -e "create database proxy_test"
[root@proxy-sql ~]# mysql -uroot -p'Zxc,./123' -P6033 -h127.0.0.1 -e "show databases;"
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+
| Database           |
+--------------------+
| information_schema |
| kevin              |
| mysql              |
| performance_schema |
| proxy_test         |
| sys                |
+--------------------+

# 测似乎sqlsender用户
[root@proxy-sql ~]# mysql -usqlsender -p'Zxc,./123' -P6033 -h127.0.0.1 -e 'drop database proxy_test;'
mysql: [Warning] Using a password on the command line interface can be insecure.
[root@proxy-sql ~]# mysql -usqlsender -p'Zxc,./123' -P6033 -h127.0.0.1 -e 'show databases;'
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+
| Database           |
+--------------------+
| information_schema |
| kevin              |
| mysql              |
| performance_schema |
| sys                |
+--------------------+

```
## 读写分离：配置路由规则
**在配置路由规则的情况下，mysql_query_rules的规则较mysql_replication_hostgroups优先级更高。**
查询规则有关的表有两个:mysql_query_rules和mysql_query_rules_fast_routing，后者是前者的扩展表。
本实验：插入两个规则，目的是将select语句分离到hostgroup_id=20的读组，但由于select语句中有一个特殊语句SELECT...FOR UPDATE它会申请写锁，所以应该路由到hostgroup_id=10的写组.
``` bash
# 在proxysql服务器上执行
# select ... for update规则的rule_id必须要小于普通的select规则的rule_id，因为ProxySQL是根据rule_id的顺序进行规则匹配的。
[root@proxy-sql ~]# mysql -uadmin -padmin -P6032 -h127.0.0.1
······
mysql> insert into mysql_query_rules(rule_id,active,match_digest,destination_hostgroup,apply) VALUES (1,1,'^SELECT.*FOR UPDATE$',10,1), (2,1,'^SELECT',20,1);
Query OK, 2 rows affected (0.00 sec)

mysql> load mysql query rules to runtime;
Query OK, 0 rows affected (0.00 sec)

mysql> save mysql query rules to disk;
Query OK, 0 rows affected (0.20 sec)


# 测试下，读操作是否路由给了hostgroup_id=20的读组, 如下发现server_id为52和53的节点 (即slave从节点)在读组内
[root@proxy-sql ~]# mysql -uroot -pZxc,./123 -P6033 -h127.0.0.1 -e 'select @@server_id'
mysql: [Warning] Using a password on the command line interface can be insecure.
+-------------+
| @@server_id |
+-------------+
|          53 |
+-------------+

# 读操作已经路由给读组，再看看写操作。这里以事务持久化进行测试。
[root@proxy-sql ~]# mysql -uroot -pZxc,./123 -P6033 -h127.0.0.1 -e 'start transaction;select @@server_id;commit;select @@server_id;'
mysql: [Warning] Using a password on the command line interface can be insecure.
+-------------+
| @@server_id |
+-------------+
|          51 |
+-------------+
+-------------+
| @@server_id |
+-------------+
|          53 |
+-------------+

# 这里已经能够根据路由来分配执行的库。

# 如果想查看路由的信息，可查询stats库中的stats_mysql_query_digest表
[root@proxy-sql ~]# mysql -uadmin -padmin -P6032 -h127.0.0.1
·········
mysql> SELECT hostgroup hg, sum_time, count_star, digest_text FROM stats_mysql_query_digest ORDER BY sum_time DESC;
+----+----------+------------+----------------------------------+
| hg | sum_time | count_star | digest_text                      |
+----+----------+------------+----------------------------------+
| 10 | 48464    | 1          | create database proxy_test       |
| 10 | 36289    | 1          | drop database proxy_test         |
| 20 | 29303    | 12         | select @@server_id               |
| 10 | 8506     | 2          | select @@server_id               |
| 10 | 5571     | 1          | start transaction                |
| 10 | 3305     | 1          | show databases                   |
| 10 | 2682     | 1          | show databases                   |
| 10 | 1645     | 1          | commit                           |
| 10 | 1437     | 1          | SELECT DATABASE()                |
| 10 | 0        | 15         | select @@version_comment limit ? |
| 10 | 0        | 2          | select @@version_comment limit ? |
+----+----------+------------+----------------------------------+
```

## 测试读写分离
由于读写操作都记录在proxysql的stats_mysql_query_digest表内。
为了测试读写分离的效果，可以先清空此表中之前的记录 (即之前在实现读写分配路由配置之前的记录)
``` bash
# 这个命令是专门清空stats_mysql_query_digest表的  (使用"delete from stats_mysql_query_digest"  清空不掉!)
mysql> SELECT 1 FROM stats_mysql_query_digest_reset LIMIT 1;
Empty set (0.00 sec)

mysql> SELECT * FROM stats_mysql_query_digest_reset;
Empty set (0.00 sec)

# 在mysql-proxy代理层节点，通过proxysql进行数据写入，并查看
[root@proxy-sql ~]# mysql -uroot -pZxc,./123 -P6033 -h127.0.0.1 -e 'select * from kevin.haha;'
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+----------+
| id | name     |
+----+----------+
|  1 | congcong |
|  2 | huihui   |
|  3 | grace    |
|  4 | qiufuqi  |
+----+----------+

[root@proxy-sql ~]# mysql -uroot -pZxc,./123 -P6033 -h127.0.0.1 -e 'delete from kevin.haha where id > 3;'
[root@proxy-sql ~]# mysql -uroot -pZxc,./123 -P6033 -h127.0.0.1 -e 'insert into kevin.haha values(21,"zhongguo"),(22,"xianggang"),(23,"taiwan");'
[root@proxy-sql ~]# mysql -uroot -pZxc,./123 -P6033 -h127.0.0.1 -e 'update kevin.haha set name="hangzhou" where id=22 ;'                
[root@proxy-sql ~]# mysql -uroot -pZxc,./123 -P6033 -h127.0.0.1 -e 'select * from kevin.haha;'    
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+----------+
| id | name     |
+----+----------+
|  1 | congcong |
|  2 | huihui   |
|  3 | grace    |
| 21 | zhongguo |
| 22 | hangzhou |
| 23 | taiwan   |
+----+----------+

# 经过查验，发现数据已经写到mysql-master主数据库上，并同步到mysql-slave1和mysql-slave2两个从数据库上了！

# 在proxysql管理端查看读写分离
[root@proxy-sql ~]#  mysql -uadmin -padmin -h127.0.0.1 -P6032
·········
mysql> select hostgroup,username,digest_text,count_star from stats_mysql_query_digest;
+-----------+----------+------------------------------------------------+------------+
| hostgroup | username | digest_text                                    | count_star |
+-----------+----------+------------------------------------------------+------------+
| 10        | root     | insert into kevin.haha values(?,?),(?,?),(?,?) | 1          |
| 10        | root     | delete from kevin.haha where id > ?            | 1          |
| 10        | root     | update kevin.haha set name=? where id=?        | 1          |
| 20        | root     | select * from kevin.haha                       | 3          |
| 10        | root     | select @@version_comment limit ?               | 6          |
+-----------+----------+------------------------------------------------+------------+
5 rows in set (0.00 sec)

# 读写分离配置是成功的，读请求是转发到group 20的读组内，写请求转发到group 10的写组内!!
```

## ProxySQL的Web统计功能
``` bash
# 打开web功能
[root@proxy-sql ~]# mysql -uadmin -padmin -h127.0.0.1 -P6032
·········
mysql> update global_variables set variable_value='true' where variable_name='admin-web_enabled';
Query OK, 1 row affected (0.01 sec)

mysql> load admin variables to runtime;
Query OK, 0 rows affected (0.00 sec)

mysql> save admin variables to disk;
Query OK, 31 rows affected (0.11 sec)

# 查看端口和登录web界面的用户名和密码，用户名和密码与stat账户一致
mysql> select * from global_variables where variable_name LIKE 'admin-web%' or variable_name LIKE 'admin-stats%';
+-----------------------------------+------------------+
| variable_name                     | variable_value   |
+-----------------------------------+------------------+
| admin-stats_credentials           | statsuser:123456 |
| admin-stats_mysql_connections     | 60               |
| admin-stats_mysql_connection_pool | 60               |
| admin-stats_mysql_query_cache     | 60               |
| admin-stats_system_cpu            | 60               |
| admin-stats_system_memory         | 60               |
| admin-web_enabled                 | true             |
| admin-web_port                    | 6080             |
+-----------------------------------+------------------+
8 rows in set (0.00 sec)
```
开放6080端口，浏览器访问http://10.128.1.44:6080 并使用statsuser:123456登录即可查看一些统计信息。
![图片](https://qiufuqi.github.io/img/hexo/20220908154944.png)

## scheduler定时任务
打印proxysql状态到日志
``` bash
[root@proxy-sql ~]# mkdir -p /opt/proxysql/log
[root@proxy-sql ~]# vi /opt/proxysql/log/status.sh
#!/bin/bash
DATE=`date "+%Y-%m-%d %H:%M:%S"`
echo "{\"dateTime\":\"$DATE\",\"status\":\"running\"}" >> /opt/proxysql/log/status_log

[root@proxy-sql ~]# chmod 777 /opt/proxysql/log/status.sh

# 在proxysql插入一条scheduler (定义每分钟打印一次，即60000毫秒)
[root@proxy-sql ~]# mysql -uadmin -padmin -h127.0.0.1 -P6032
·········
mysql> insert into scheduler(active,interval_ms,filename) values (1,60000,'/opt/proxysql/log/status.sh');
Query OK, 1 row affected (0.01 sec)

mysql> LOAD SCHEDULER TO RUNTIME;
Query OK, 0 rows affected (0.00 sec)

mysql> SAVE SCHEDULER TO DISK;
Query OK, 0 rows affected (0.13 sec)

# 查看日志就可以看到proxysql 的运行结果了
[root@proxy-sql ~]# tail -f /opt/proxysql/log/status_log
{"dateTime":"2022-09-08 15:54:57","status":"running"}
{"dateTime":"2022-09-08 15:55:57","status":"running"}

```