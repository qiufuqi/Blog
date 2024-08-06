---
title: ProxySQL+MGR读写分离
date: 2022-09-09
tags:
  - Linux
  - Mysql
  - ProxySQL
categories: 
- 运维
- 数据库
- ProxySQL
- MGR同步
keywords: 'Linux,Mysql,ProxySQL,MGR'
description: ProxySQL+MGR读写分离
cover: https://qiufuqi.github.io/img/hexo/20220909115022.png
abbrlink: ProxySQL_MGR
comments: false
---

ProxySQL实现MGR读写分离

实现思路：三个节点使multi-primary的方式连接，应用通过连接ProxySQL中间件，根据sql的属性（是否为select语句）来决定连接哪一个节点，一个可写节点，两个只读节点（其实三个都是可写节点，只不过通过proxysql进行了读写分离）。如果默认的可写节点挂掉的话，proxysql通过定期运行的调度器会将另一个只读节点的其中一台设为可写节点，实际主节点故障应用无感应的要求。上述的整个过程中，应用无需任何变动。应用从意识发生了故障，到连接重新指向新的主，正常提供服务，秒级别的间隔。
本实验前期准备：
- [ProxySQL中间件](ProxySQL) 
- [Mysql5.7 MGR多写模式](/mysql5.7_MGR)
  
![](https://qiufuqi.github.io/img/hexo/20220909115022.png)

**ProxySQL实现MGR读写分离** [参考地址](https://www.cnblogs.com/kevingrace/p/10384691.html)

## 环境准备
四台服务器 10.128.1.51，10.128.1.52，10.128.1.53已实现mysql MGR多住模式，[搭建参考](/mysql5.7_MGR)
``` bash
# 四台服务器
10.128.1.44   proxy-sql
10.128.1.41   MGR-node1 (master1)   mysql5.7
10.128.1.42   MGR-node2 (master2)   mysql5.7
10.128.1.43   MGR-node3 (master3)   mysql5.7
```

## 初始化Proxysql
将之前的数据都删除，如果是新安装可忽略
``` bash
[root@proxy-sql ~]# mysql -uadmin -padmin -h127.0.0.1 -P6032
·········
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

mysql> delete from scheduler ;
mysql> delete from mysql_servers;
mysql> delete from mysql_users;
mysql> delete from mysql_query_rules;
mysql> delete from mysql_group_replication_hostgroups;

# 修改后，加载到RUNTIME，并保存到disk
mysql> LOAD MYSQL VARIABLES TO RUNTIME;
mysql> SAVE MYSQL VARIABLES TO DISK;

mysql> LOAD MYSQL SERVERS TO RUNTIME;
mysql> SAVE MYSQL SERVERS TO DISK;

mysql>  LOAD MYSQL USERS TO RUNTIME;
mysql> SAVE MYSQL USERS TO DISK;

mysql> LOAD SCHEDULER TO RUNTIME;
mysql> SAVE SCHEDULER TO DISK;

mysql> LOAD MYSQL QUERY RULES TO RUNTIME;
mysql> SAVE MYSQL QUERY RULES TO DISK;

```
## 建立proxysql登录账号
在任意一个节点操作，会自动同步到其他节点
``` bash
# 在10.128.1.41上执行
[root@mgr-node1 ~]# mysql -uroot -p
·········
mysql> CREATE USER 'proxysql'@'%' IDENTIFIED BY 'Zxc,./123';
Query OK, 0 rows affected (0.07 sec)

mysql> GRANT ALL ON * . * TO  'proxysql'@'%';
Query OK, 0 rows affected (0.06 sec)

mysql> create user 'sbuser'@'%' IDENTIFIED BY 'Zxc,./123';
Query OK, 0 rows affected (0.04 sec)

mysql> GRANT ALL ON * . * TO 'sbuser'@'%';
Query OK, 0 rows affected (0.06 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.10 sec)

```
## 创建MGR节点状态函数和试图
在三个MGR任意一个节点上操作，会自动同步到其他节点
``` bash
在MGR-node1节点上，创建系统视图sys.gr_member_routing_candidate_status，该视图将为ProxySQL提供组复制相关的监控状态指标。
创建addition_to_sys.sql脚本，在MGR-node1节点执行如下语句导入MySQL即可 (在mgr-node1节点的mysql执行后，会同步到其他两个节点上)。

[root@mgr-node1 ~]# vi /root/addition_to_sys.sql
USE sys;
  
DELIMITER $$
  
CREATE FUNCTION IFZERO(a INT, b INT)
RETURNS INT
DETERMINISTIC
RETURN IF(a = 0, b, a)$$
  
CREATE FUNCTION LOCATE2(needle TEXT(10000), haystack TEXT(10000), offset INT)
RETURNS INT
DETERMINISTIC
RETURN IFZERO(LOCATE(needle, haystack, offset), LENGTH(haystack) + 1)$$
  
CREATE FUNCTION GTID_NORMALIZE(g TEXT(10000))
RETURNS TEXT(10000)
DETERMINISTIC
RETURN GTID_SUBTRACT(g, '')$$
  
CREATE FUNCTION GTID_COUNT(gtid_set TEXT(10000))
RETURNS INT
DETERMINISTIC
BEGIN
  DECLARE result BIGINT DEFAULT 0;
  DECLARE colon_pos INT;
  DECLARE next_dash_pos INT;
  DECLARE next_colon_pos INT;
  DECLARE next_comma_pos INT;
  SET gtid_set = GTID_NORMALIZE(gtid_set);
  SET colon_pos = LOCATE2(':', gtid_set, 1);
  WHILE colon_pos != LENGTH(gtid_set) + 1 DO
     SET next_dash_pos = LOCATE2('-', gtid_set, colon_pos + 1);
     SET next_colon_pos = LOCATE2(':', gtid_set, colon_pos + 1);
     SET next_comma_pos = LOCATE2(',', gtid_set, colon_pos + 1);
     IF next_dash_pos < next_colon_pos AND next_dash_pos < next_comma_pos THEN
       SET result = result +
         SUBSTR(gtid_set, next_dash_pos + 1,
                LEAST(next_colon_pos, next_comma_pos) - (next_dash_pos + 1)) -
         SUBSTR(gtid_set, colon_pos + 1, next_dash_pos - (colon_pos + 1)) + 1;
     ELSE
       SET result = result + 1;
     END IF;
     SET colon_pos = next_colon_pos;
  END WHILE;
  RETURN result;
END$$
  
CREATE FUNCTION gr_applier_queue_length()
RETURNS INT
DETERMINISTIC
BEGIN
  RETURN (SELECT sys.gtid_count( GTID_SUBTRACT( (SELECT
Received_transaction_set FROM performance_schema.replication_connection_status
WHERE Channel_name = 'group_replication_applier' ), (SELECT
@@global.GTID_EXECUTED) )));
END$$
  
CREATE FUNCTION gr_member_in_primary_partition()
RETURNS VARCHAR(3)
DETERMINISTIC
BEGIN
  RETURN (SELECT IF( MEMBER_STATE='ONLINE' AND ((SELECT COUNT(*) FROM
performance_schema.replication_group_members WHERE MEMBER_STATE != 'ONLINE') >=
((SELECT COUNT(*) FROM performance_schema.replication_group_members)/2) = 0),
'YES', 'NO' ) FROM performance_schema.replication_group_members JOIN
performance_schema.replication_group_member_stats USING(member_id));
END$$
  
CREATE VIEW gr_member_routing_candidate_status AS SELECT
sys.gr_member_in_primary_partition() as viable_candidate,
IF( (SELECT (SELECT GROUP_CONCAT(variable_value) FROM
performance_schema.global_variables WHERE variable_name IN ('read_only',
'super_read_only')) != 'OFF,OFF'), 'YES', 'NO') as read_only,
sys.gr_applier_queue_length() as transactions_behind, Count_Transactions_in_queue as 'transactions_to_cert' from performance_schema.replication_group_member_stats;$$
  
DELIMITER ;
###########################################

# 导入addition_to_sys.sql文件
[root@mgr-node1 ~]# mysql -uroot -p123456 < /root/addition_to_sys.sql

[root@mgr-node1 ~]# mysql -uroot -p123456
·········
mysql> select * from sys.gr_member_routing_candidate_status;
+------------------+-----------+---------------------+----------------------+
| viable_candidate | read_only | transactions_behind | transactions_to_cert |
+------------------+-----------+---------------------+----------------------+
| YES              | NO        |                   0 |                    0 |
+------------------+-----------+---------------------+----------------------+
1 row in set (0.00 sec)

```
## 在proxysql中添加账号
``` bash
[root@proxy-sql ~]# mysql -uadmin -padmin -h127.0.0.1 -P6032
·········
mysql> INSERT INTO MySQL_users(username,password,default_hostgroup) VALUES ('proxysql','Zxc,./123',1);
Query OK, 1 row affected (0.00 sec)

mysql> UPDATE global_variables SET variable_value='proxysql' where variable_name='mysql-monitor_username'; 
Query OK, 1 row affected (0.01 sec)

mysql> UPDATE global_variables SET variable_value='Zxc,./123' where variable_name='mysql-monitor_password';
Query OK, 1 row affected (0.00 sec)

# 更改后，加载到RUNTIME，并保存到disk
mysql> LOAD MYSQL SERVERS TO RUNTIME;
Query OK, 0 rows affected (0.00 sec)

mysql> SAVE MYSQL SERVERS TO DISK;
Query OK, 0 rows affected (0.31 sec)

# 测试能否登录数据库  报错的情况下
[root@proxy-sql ~]# mysql -uproxysql -pZxc,./123 -h 127.0.0.1 -P6033 -e "select @@hostname"
mysql: [Warning] Using a password on the command line interface can be insecure.
ERROR 1045 (28000): ProxySQL Error: Access denied for user 'proxysql'@'127.0.0.1' (using password: YES)

# 检查发现，明明用户名和密码已经修改成proxysql:proxysql了
mysql> select * from global_variables;
·········
| mysql-interfaces                                    | 0.0.0.0:6033                |
| mysql-default_schema                                | information_schema          |
| mysql-stacksize                                     | 1048576                     |
| mysql-server_version                                | 5.5.30                      |
| mysql-connect_timeout_server                        | 3000                        |
| mysql-monitor_username                              | proxysql                    |
| mysql-monitor_password                              | Zxc,./123                   |
| mysql-monitor_history                               | 600000                      |
| mysql-monitor_connect_interval                      | 60000                       |
| mysql-monitor_ping_interval                         | 10000                       |
·········

# 解决方法
mysql> LOAD MYSQL VARIABLES TO RUNTIME;
mysql> SAVE MYSQL VARIABLES TO DISK;

mysql> LOAD MYSQL SERVERS TO RUNTIME;
mysql> SAVE MYSQL SERVERS TO DISK;

mysql> LOAD MYSQL USERS TO RUNTIME;
mysql> SAVE MYSQL USERS TO DISK;

mysql> LOAD SCHEDULER TO RUNTIME;
mysql> SAVE SCHEDULER TO DISK;

mysql> LOAD MYSQL QUERY RULES TO RUNTIME;
mysql> SAVE MYSQL QUERY RULES TO DISK;

# 再次验证 因为后端三个mysql的MGR节点还没有加入到proxysql中的原因
[root@proxy-sql ~]# mysql -uproxysql -pZxc,./123 -h 127.0.0.1 -P6033 -e "select @@hostname"
ERROR 9001 (HY000) at line 1: Max connect timeout reached while reaching hostgroup 1 after 10000ms

```
## 配置proxysql
添加mysql节点，并分组。
``` bash
[root@proxy-sql ~]# mysql -uadmin -padmin -h127.0.0.1 -P6032
·········
mysql> delete from mysql_servers;
Query OK, 1 row affected (0.00 sec)

mysql> insert into mysql_servers (hostgroup_id, hostname, port) values(1,'10.128.1.41',3306);
mysql> insert into mysql_servers (hostgroup_id, hostname, port) values(1,'10.128.1.42',3306);
mysql> insert into mysql_servers (hostgroup_id, hostname, port) values(1,'10.128.1.43',3306);

mysql> insert into mysql_servers (hostgroup_id, hostname, port) values(2,'10.128.1.41',3306);
mysql> insert into mysql_servers (hostgroup_id, hostname, port) values(2,'10.128.1.42',3306);
mysql> insert into mysql_servers (hostgroup_id, hostname, port) values(2,'10.128.1.43',3306);

mysql> select * from  mysql_servers ;
+--------------+-------------+------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| hostgroup_id | hostname    | port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
+--------------+-------------+------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| 1            | 10.128.1.41 | 3306 | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 1            | 10.128.1.42 | 3306 | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 1            | 10.128.1.43 | 3306 | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 2            | 10.128.1.41 | 3306 | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 2            | 10.128.1.42 | 3306 | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 2            | 10.128.1.43 | 3306 | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
+--------------+-------------+------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+

hostgroup_id = 1代表write group，针对我们提出的限制，这个地方只配置了一个节点；
hostgroup_id = 2代表read group，包含了MGR的所有节点,目前只是Onlinle的，等配置过scheduler后，status就会有变化 。
  
对于上面的hostgroup配置，默认所有的写操作会发送到hostgroup_id为1的online节点，也就是发送到写节点上。
所有的读操作，会发送为hostgroup_id为2的online节点。
  
需要确认一下没有使用proxysql的读写分离规则（因为之前测试中配置了这个地方，所以需要删除，以免影响后面的测试）。
mysql> delete from mysql_query_rules;
Query OK, 0 rows affected (0.00 sec)

mysql> commit;
Query OK, 0 rows affected (0.00 sec)

# 需要将global_variables，mysql_servers、mysql_users表的信息加载到RUNTIME，更进一步加载到DISK:
mysql> LOAD MYSQL VARIABLES TO RUNTIME;
mysql> SAVE MYSQL VARIABLES TO DISK;

mysql> LOAD MYSQL SERVERS TO RUNTIME;
mysql> SAVE MYSQL SERVERS TO DISK;

mysql> LOAD MYSQL USERS TO RUNTIME;
mysql> SAVE MYSQL USERS TO DISK;

# 再次验证proxysql登录
[root@proxy-sql ~]# mysql -uproxysql -pZxc,./123 -h 127.0.0.1 -P6033 -e "select @@hostname"
mysql: [Warning] Using a password on the command line interface can be insecure.
+------------+
| @@hostname |
+------------+
| mgr-node2  |
+------------+
[root@proxy-sql ~]# mysql -uproxysql -pZxc,./123 -h 127.0.0.1 -P6033 -e "select @@hostname"
mysql: [Warning] Using a password on the command line interface can be insecure.
+------------+
| @@hostname |
+------------+
| mgr-node1  |
+------------+
```
## 配置scheduler
首先去github下载对应脚本，[下载地址](https://github.com/ZzzCrazyPig/proxysql_groupreplication_checker)
共有三个脚本可供下载：
- proxysql_groupreplication_checker.sh：用于multi-primary模式，可以实现读写分离，以及故障切换，同一时间点多个节点可以多写；
- gr_mw_mode_cheker.sh：用于multi-primary模式，可以实现读写分离，以及故障切换，不过在同一时间点只能有一个节点能写；
- gr_sw_mode_checker.sh：用于single-primary模式，可以实现读写分离，以及故障切换；

实验的环境是multi-primary模式，所以选择proxysql_groupreplication_checker.sh脚本
``` bash
百度云盘上，下载地址：https://pan.baidu.com/s/1lUzr58BSA_U7wmYwsRcvzQ
提取密码：9rm7

# proxysql_groupreplication_checker.sh放到目录/var/lib/proxysql/下，并增加可以执行的权限
[root@proxy-sql ~]# chmod a+x /var/lib/proxysql/proxysql_groupreplication_checker.sh
[root@proxy-sql ~]# ll /var/lib/proxysql/proxysql_groupreplication_checker.sh
-rwxr-xr-x. 1 root root 6081 Sep  9 15:22 /var/lib/proxysql/proxysql_groupreplication_checker.sh

# 在proxysql的scheduler表里面添加记录，加载到runtime，同时还可以持久化到磁盘:
[root@proxy-sql ~]# mysql -uadmin -padmin -h127.0.0.1 -P6032
·········
mysql> INSERT INTO scheduler(id,interval_ms,filename,arg1,arg2,arg3,arg4, arg5) VALUES (1,'10000','/var/lib/proxysql/proxysql_groupreplication_checker.sh','1','2','1','0','/var/lib/proxysql/proxysql_groupreplication_checker.log');
Query OK, 1 row affected (0.00 sec)

mysql> select * from scheduler;
+----+--------+-------------+--------------------------------------------------------+------+------+------+------+---------------------------------------------------------+---------+
| id | active | interval_ms | filename                                               | arg1 | arg2 | arg3 | arg4 | arg5                                                    | comment |
+----+--------+-------------+--------------------------------------------------------+------+------+------+------+---------------------------------------------------------+---------+
| 1  | 1      | 10000       | /var/lib/proxysql/proxysql_groupreplication_checker.sh | 1    | 2    | 1    | 0    | /var/lib/proxysql/proxysql_groupreplication_checker.log |         |
+----+--------+-------------+--------------------------------------------------------+------+------+------+------+---------------------------------------------------------+---------+
1 row in set (0.00 sec)

mysql> LOAD SCHEDULER TO RUNTIME;
mysql> SAVE SCHEDULER TO DISK;

==============================================================================
scheduler各column的说明：
active : 1: enable scheduler to schedule the script we provide
interval_ms : invoke one by one in cycle (eg: 5000(ms) = 5s represent every 5s invoke the script)
filename: represent the script file path
arg1~arg5: represent the input parameters the script received
 
脚本proxysql_groupreplication_checker.sh对应的参数说明如下:
arg1 is the hostgroup_id for write  可写入组
arg2 is the hostgroup_id for read   可读取组
arg3 is the number of writers we want active at the same time   同时写入组
arg4 represents if we want that the member acting for writes is also candidate for reads 可写的节点是否用户可读
arg5 is the log file


# schedule信息加载后，就会分析当前的环境，mysql_servers中显示出当前只有10.128.1.41是可以写的，
# 10.128.1.42以及10.128.1.43是用来读的。
mysql> select * from  mysql_servers ;
+--------------+-------------+------+--------------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| hostgroup_id | hostname    | port | status       | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
+--------------+-------------+------+--------------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| 1            | 10.128.1.41 | 3306 | ONLINE       | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 1            | 10.128.1.42 | 3306 | OFFLINE_SOFT | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 1            | 10.128.1.43 | 3306 | OFFLINE_SOFT | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 2            | 10.128.1.41 | 3306 | OFFLINE_SOFT | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 2            | 10.128.1.42 | 3306 | ONLINE       | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 2            | 10.128.1.43 | 3306 | ONLINE       | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
+--------------+-------------+------+--------------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
6 rows in set (0.00 sec)


# 因为schedule的arg4，我这里设为了0，就表示可写的节点不能用于读。那我将arg4设置为1试一下：
mysql> update scheduler set arg4=1;

mysql> LOAD SCHEDULER TO RUNTIME;
mysql> SAVE SCHEDULER TO DISK;

mysql> select * from  mysql_servers;
+--------------+-------------+------+--------------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| hostgroup_id | hostname    | port | status       | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
+--------------+-------------+------+--------------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| 1            | 10.128.1.41 | 3306 | ONLINE       | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 1            | 10.128.1.42 | 3306 | OFFLINE_SOFT | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 1            | 10.128.1.43 | 3306 | OFFLINE_SOFT | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 2            | 10.128.1.41 | 3306 | ONLINE       | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 2            | 10.128.1.42 | 3306 | ONLINE       | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 2            | 10.128.1.43 | 3306 | ONLINE       | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
+--------------+-------------+------+--------------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
6 rows in set (0.00 sec)

arg4设置为1之后，10.128.1.41节点用来写的同时，也可以被用来读。

# 便于下面的测试还是将arg4设为0：
mysql> update scheduler set arg4=0;

mysql> LOAD SCHEDULER TO RUNTIME;
mysql> SAVE SCHEDULER TO DISK;

# 各个节点的gr_member_routing_candidate_status视图也显示了当前节点是否是正常状态的，
# proxysql就是读取的这个视图的信息来决定此节点是否可用。
[root@mgr-node1 ~]# mysql -uroot -p
·········
mysql> select * from sys.gr_member_routing_candidate_status\G;
*************************** 1. row ***************************
    viable_candidate: YES
           read_only: NO
 transactions_behind: 0
transactions_to_cert: 0
1 row in set (0.01 sec)

ERROR: 
No query specified
```

## 设置读写分离规则
### 读写分离规则配置
查询规则有关的表有两个:mysql_query_rules和mysql_query_rules_fast_routing，后者是前者的扩展表。
``` bash
[root@proxy-sql ~]# mysql -uadmin -padmin -P6032 -h127.0.0.1
·········
# select ... for update规则的rule_id必须要小于普通的select规则的rule_id，因为ProxySQL是根据rule_id的顺序进行规则匹配的。
mysql> insert into mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply) values (3, 1,"^SELECT",2,1),(4, 1,"^SELECT.*FOR UPDATE$",1,1);
Query OK, 1 row affected (0.00 sec)

mysql> LOAD MYSQL QUERY RULES TO RUNTIME;
mysql> SAVE MYSQL QUERY RULES TO DISK;

rule_id ProxySQL是根据rule_id的顺序进行规则匹配的。
active表示是否启用这个sql路由项，
match_pattern的规则是基于正则表达式的，
destination_hostgroup表示我们要将该类sql转发到哪些mysql上面去，这里我们将select转发到group 2，select for update转发到group 1。
apply为1表示该正则匹配后，将不再接受其他匹配，直接转发。

# 在proxysql本机或其他客户机上检查下，select 语句，一直连接的是10.128.1.42和10.128.1.43
[root@mgr-node1 ~]# mysql -uproxysql -pZxc,./123 -h10.128.1.44 -P6033 -e "select @@hostname"
mysql: [Warning] Using a password on the command line interface can be insecure.
+------------+
| @@hostname |
+------------+
| mgr-node2  |
+------------+
```
### 验证读写分离规则
``` bash
[root@mgr-node1 ~]# mysql -uproxysql -pZxc,./123 -h10.128.1.44 -P6033 -e "select @@hostname"
mysql: [Warning] Using a password on the command line interface can be insecure.
+------------+
| @@hostname |
+------------+
| mgr-node2  |
+------------+

[root@mgr-node1 ~]# mysql -uproxysql -pZxc,./123 -h10.128.1.44 -P6033 -e "select * from kaka.kaka_test"
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+-------+
| id | name  |
+----+-------+
|  1 | 223   |
|  2 | 345   |
| 15 | 44    |
| 22 | 444   |
| 23 | 344   |
| 30 | 555   |
| 37 | rtr   |
| 44 | dsfda |
| 51 | dafd  |
| 58 | adsfd |
+----+-------+

[root@mgr-node1 ~]# mysql -uproxysql -pZxc,./123 -h10.128.1.44 -P6033 -e "delete from kaka.kaka_test where id > 25;"
[root@mgr-node1 ~]# mysql -uproxysql -pZxc,./123 -h10.128.1.44 -P6033 -e "delete from kaka.kaka_test where id=2;"
[root@mgr-node1 ~]# mysql -uproxysql -pZxc,./123 -h10.128.1.44 -P6033 -e "select * from kaka.kaka_test"
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+------+
| id | name |
+----+------+
|  1 | 223  |
| 15 | 44   |
| 22 | 444  |
| 23 | 344  |
+----+------+

[root@mgr-node1 ~]# mysql -uproxysql -pZxc,./123 -h10.128.1.44 -P6033 -e 'insert into kaka.kaka_test values(31,"zhongguo"),(32,"xianggang"),(33,"taiwan");'
[root@mgr-node1 ~]# mysql -uproxysql -pZxc,./123 -h10.128.1.44 -P6033 -e "select * from kaka.kaka_test"  
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+-----------+
| id | name      |
+----+-----------+
|  1 | 223       |
| 15 | 44        |
| 22 | 444       |
| 23 | 344       |
| 31 | zhongguo  |
| 32 | xianggang |
| 33 | taiwan    |
+----+-----------+

# 最后在proxysql管理端查看读写分离情况
[root@proxy-sql ~]# mysql -uadmin -padmin -P6032 -h127.0.0.1
·········
mysql> select hostgroup,username,digest_text,count_star from stats_mysql_query_digest;
+-----------+----------+----------------------------------------------------+------------+
| hostgroup | username | digest_text                                        | count_star |
+-----------+----------+----------------------------------------------------+------------+
| 1         | proxysql | insert into kaka.kaka_test values(?,?),(?,?),(?,?) | 1          |
| 1         | proxysql | insert into kevin.hah values(?,?),(?,?),(?,?)      | 1          |
| 1         | proxysql | insert into kevin.haha values(?,?),(?,?),(?,?)     | 1          |
| 1         | proxysql | delete from kaka.kaka_test where id=?              | 1          |
| 1         | proxysql | delete from kaka.kaka_test where id > ?            | 1          |
| 2         | proxysql | select * from kaka.kaka_test                       | 3          |
| 2         | proxysql | select * from kevin.haha                           | 1          |
| 2         | proxysql | select @@hostname                                  | 4          |
| 1         | proxysql | select @@hostname                                  | 3          |
| 1         | proxysql | select @@version_comment limit ?                   | 16         |
+-----------+----------+----------------------------------------------------+------------+

mysql> select * from  mysql_servers;
+--------------+-------------+------+--------------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| hostgroup_id | hostname    | port | status       | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
+--------------+-------------+------+--------------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| 1            | 10.128.1.41 | 3306 | ONLINE       | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 1            | 10.128.1.42 | 3306 | OFFLINE_SOFT | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 1            | 10.128.1.43 | 3306 | OFFLINE_SOFT | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 2            | 10.128.1.41 | 3306 | OFFLINE_SOFT | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 2            | 10.128.1.42 | 3306 | ONLINE       | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 2            | 10.128.1.43 | 3306 | ONLINE       | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
+--------------+-------------+------+--------------+--------+-------------+-----------------+---------------------+---------+----------------+---------+

# 写操作都分配到了group1组内，即写操作分配到10.128.1.41节点上。
# 读操作都分配到了group2组内，即读操作分配到10.128.1.42、10.128.1.43节点上。

```
## 设置故障应用无感应
在上面的读写分离规则中，我设置了10.128.1.41为可写节点，10.128.1.42,10.128.1.43为只读节点
如果此时10.128.1.41变成只读模式的话，应用能不能直接连到其它的节点进行写操作？
``` bash
# 现在手动将10.128.1.41变成只读模式
[root@mgr-node1 ~]# mysql -uroot -p
·········
mysql> set global read_only=1;
Query OK, 0 rows affected (0.00 sec)

# 接着观察一下mysql_servers的状态，自动将group1的10.128.1.42改成了online，group2的10.128.1.41,10.128.1.43变成online了
# 表示将10.128.1.42变为可写节点，其它两个节点变为只读节点了。
[root@proxy-sql ~]# mysql -uadmin -padmin -P6032 -h127.0.0.1
·········
mysql> select * from  mysql_servers;
+--------------+-------------+------+--------------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| hostgroup_id | hostname    | port | status       | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
+--------------+-------------+------+--------------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| 1            | 10.128.1.41 | 3306 | OFFLINE_SOFT | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 1            | 10.128.1.42 | 3306 | ONLINE       | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 1            | 10.128.1.43 | 3306 | OFFLINE_SOFT | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 2            | 10.128.1.41 | 3306 | ONLINE       | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 2            | 10.128.1.42 | 3306 | OFFLINE_SOFT | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 2            | 10.128.1.43 | 3306 | ONLINE       | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
+--------------+-------------+------+--------------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
6 rows in set (0.00 sec)
# 可进行读写测试

# 再次将10.128.1.41变为可写模式后,mysql_servers也恢复过来了。
[root@mgr-node1 ~]# mysql -uroot -p
·········
mysql> set global read_only=0;
Query OK, 0 rows affected (0.00 sec)


[root@proxy-sql ~]# mysql -uadmin -padmin -P6032 -h127.0.0.1
·········
mysql> select * from  mysql_servers;
+--------------+-------------+------+--------------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| hostgroup_id | hostname    | port | status       | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
+--------------+-------------+------+--------------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| 1            | 10.128.1.41 | 3306 | ONLINE       | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 1            | 10.128.1.42 | 3306 | OFFLINE_SOFT | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 1            | 10.128.1.43 | 3306 | OFFLINE_SOFT | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 2            | 10.128.1.41 | 3306 | OFFLINE_SOFT | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 2            | 10.128.1.42 | 3306 | ONLINE       | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 2            | 10.128.1.43 | 3306 | ONLINE       | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
+--------------+-------------+------+--------------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
6 rows in set (0.00 sec)

```
经过测试将10.128.1.41节点停止组复制（stop group_replication）或者该节点宕机(mysql服务挂掉)后，mysql_servers表的信息也会正常的切换新的节点。
待10.128.1.41恢复再加入到组复制后，mysql_servers也会正常的将10.128.1.41改成online状态。

可能出现问题，所有节点都offline了的情况，查看错误日志如下
``` bash
# 查看日志
[root@proxy-sql ~]# tail -f /var/lib/proxysql/proxysql.log

```

## 监控配置
``` bash
#查看监控配置
select * from global_variables where variable_name like 'mysql-monitor%';
#检查ping
select * FROM monitor.mysql_server_ping_log ORDER BY time_start_us DESC LIMIT 10;
#mysql_server_connect_log
select * FROM monitor.mysql_server_connect_log ORDER BY time_start_us DESC LIMIT 6;


# 可以修改监控时间间隔
UPDATE global_variables SET variable_value='2000' WHERE variable_name
IN ('mysql-monitor_connect_interval','mysqlmonitor_ping_interval','mysql-monitor_read_only_interval');
```

到此，ProxySQL就简单实现了MGR的读写分离和主节点故障无感知环境。


