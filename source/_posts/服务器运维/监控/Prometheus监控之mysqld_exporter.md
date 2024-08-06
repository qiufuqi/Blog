---
title: Prometheus监控之mysqld_exporter
date: 2023-12-07
tags:
  - Prometheus
  - Mysql
categories: 
- 运维
- 监控
- Mysql
keywords: 'Prometheus,中间件,Mysql'
description: Prometheus监控,Mysql
cover: https://qiufuqi.github.io/img/hexo/20231213151555.png
abbrlink: prometheus_mysql
url: prometheus_mysql
comments: false
top: 998
---
目前使用到的中间件有 Nginx、Redis、RabbitMQ、MySql 等，介绍怎样使用 Promtheus 来监控这些中间件。[本文参考](https://mp.weixin.qq.com/s/RlUvuBcgQJTK6b0V8-jDew)
Prometheus 的数据走向，如下：
![](https://qiufuqi.github.io/img/hexo/20231205114030.png)
从图中可以看出，监控中间件的第一步就是安装中间件的 exporter，安装有两种方式：下载安装文件进行安装和使用 Docker 进行安装，[Prometheus安装参考](/prometheus)。
推荐使用文件安装，不用docker，因为docker存在有部分属性可能无法获取问题。
**Mysql监控**

# 添加Mysql账户
进入mysql命令行环境，运行一下命令，创建exporter用户，用户名密码可根据自己需要修改。
建议为该用户设置最大连接数限制，以避免因监控数据抓取对数据库带来影响
```bash
[root@localhost ~]# mysql -uroot -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
·········
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

CREATE USER 'exporter'@'%' IDENTIFIED BY 'Aabb&111';
GRANT PROCESS, REPLICATION CLIENT ON *.* TO 'exporter'@'%';
GRANT SELECT ON performance_schema.* TO 'exporter'@'%';
flush privileges;
```

# mysqld_exporter安装部署
## 脚本安装mysqld_exporter
mysqld_exporter.sh一键监控安装脚本，提前下载好安装文件或者[在线下载](https://github.com/prometheus/mysqld_exporter/releases)
```bash
#!/bin/bash
# -*- coding: utf-8 -*-
# Date: 2023/12/13
echo "download mysqld_exporter"
sleep 2
wget -N -P /root/ https://github.com/prometheus/mysqld_exporter/releases/download/v0.15.1/mysqld_exporter-0.15.1.linux-amd64.tar.gz
 

echo "tar mysqld_exporter"
sleep 2
tar -zxvf /root/mysqld_exporter-0.15.1.linux-amd64.tar.gz -C /opt/ && mv /opt/mysqld_exporter-0.15.1.linux-amd64 /usr/local/mysqld_exporter

echo "delete mysqld_exporter***tar.gz"
rm -rf /root/mysqld_exporter-0.15.1.linux-amd64.tar.gz

echo "config mysqld_exporter"
sleep 2
cat << EOF > /var/lib/mysql/.my.cnf
[client]
host=127.0.0.1
port=3306
user=exporter
password=Aabb&111
EOF
 
echo "firewall mysqld_exporter port 9104"
sleep 2
firewall-cmd --zone=public --add-port=9104/tcp --permanent && firewall-cmd --reload 
 
echo "add mysqld_exporter.service"
sleep 2
cat << EOF > /usr/lib/systemd/system/mysqld_exporter.service
[Unit]
Description=mysqld_exporter
After=network.target 
 
[Service]
Restart=on-failure
ExecStart=/usr/local/mysqld_exporter/mysqld_exporter --web.listen-address=:9104 --config.my-cnf=/var/lib/mysql/.my.cnf 

[Install]
WantedBy=multi-user.target
EOF
 
echo "start mysqld_exporter.service"
sleep 2
systemctl daemon-reload && systemctl start mysqld_exporter && systemctl enable --now mysqld_exporter
```


## 逐步安装

### 创建.my.cnf文件
在目录 /var/lib/mysql 中创建 .my.cnf 文件，文件内容如下
- host 配置为 mysql 数据库的 IP （本机的话可以填写127.0.0.1）
- user 和 password 配置为新创建的账号和密码
  
```bash
[root@localhost ~]# vi /var/lib/mysql/.my.cnf
[client]
# 部分版本不支持，可填写本机IP，比如10.11.8.108
host=127.0.0.1
port=3306
user=exporter
password=Aabb&111
```
### 安装mysqld-exporter
有两种方式：文件安装和docker安装，其他编译安装暂时不提 
#### 文件安装exporter
[文件下载地址1](https://github.com/prometheus/mysqld_exporter/releases)
[文件下载地址2](https://prometheus.io/download/#mysqld_exporter)
安装mysqld_exporter，采集机器运行数据信息，默认端口9104 （可更改为指定端口）
```bash
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.15.0/mysqld_exporter-0.15.0.linux-amd64.tar.gz

# 解压至指定文件夹
mkdir /usr/local/mysqld_exporter
tar -zxvf mysqld_exporter-0.15.0.linux-amd64.tar.gz -C /usr/local/mysqld_exporter

cd /usr/local/mysqld_exporter
cp mysqld_exporter-0.15.0.linux-amd64/* . && rm -rf mysqld_exporter-0.15.0.linux-amd64

### 验证配置是否有问题：运行以下命令，并同时访问http://10.11.8.108:9104/
/usr/local/mysqld_exporter/mysqld_exporter --web.listen-address=:9104 --config.my-cnf=/var/lib/mysql/.my.cnf
# 可能会报错，一般是mysql开通的exporter问题
ts=2023-12-08T00:28:05.034Z caller=exporter.go:152 level=error msg="Error pinging mysqld" err="Error 1045 (28000): Access denied for user 'exporter'@'localhost' (using password: YES)"



# 配置node_exporter开机自启
vi /usr/lib/systemd/system/mysqld_exporter.service
# 写入以下信息：
[Unit]
Description=mysqld_exporter
After=network.target 
 
[Service]
Restart=on-failure
ExecStart=/usr/local/mysqld_exporter/mysqld_exporter --web.listen-address=:9104 --config.my-cnf=/var/lib/mysql/.my.cnf 

[Install]
WantedBy=multi-user.target

#重启服务
systemctl daemon-reload
systemctl start mysqld_exporter
systemctl status mysqld_exporter
systemctl enable mysqld_exporter
```

#### docker安装exporter
执行以下命令，部署mysqld-exporter
```bash
docker pull prom/mysqld-exporter
docker run -d -p 9104:9104 --name mysqld-exporter --restart="always" -v /var/lib/mysql/.my.cnf:/.my.cnf prom/mysqld-exporter
```

# 访问mysqld-exporter
浏览器运行访问 http://10.11.8.108:9104/
![](https://qiufuqi.github.io/img/hexo/20231207153108.png)

# 加入prometheus监控
## 静态部署
登录prometheus所在服务器，在文件的最下面添加job 配置，并重启Prometheus
```bash
[root@prometheus ~]# vi /usr/local/prometheus/prometheus.yml
······
  - job_name: 'Mysql'
    scrape_interval: 5s
    static_configs:
      - targets: ['10.11.7.29:9104','10.11.7.34:9104','10.11.7.55:9104','10.11.7.71:9104','10.11.7.87:9104']
······
# 重启Prometheus服务，并访问prometheus所在的target状态，如果为UP说明job设置成功：
[root@prometheus ~]# cd /usr/local/prometheus
[root@prometheus prometheus]# ./promtool check config prometheus.yml
[root@prometheus ~]# systemctl restart prometheus
# 保持热加载
[root@prometheus ~]# curl  -X POST localhost:9090/-/reload
```
## 文件部署
当有新的节点时，只需要修改对应的yml或者json文件即可。
```bash
vi /usr/local/prometheus/prometheus.yml
······
  # 文件部署
  - job_name: 'Mysql'
    scrape_interval: 5s
    file_sd_configs: # 基于文件服务发现
      - files:
        - "./device/mysql_device.yml"   # - "./device/mysql_device.json"  yml和json格式选择一个即可
        refresh_interval: 5s # 每隔5秒检查刷新一次服务发现配置


 vi /usr/local/prometheus/device/mysql_device.yml
 # 或者单独列
- labels:
    desc: mysql     # 无实际意义
  targets:
    - '10.11.7.29:9104'
    - '10.11.7.34:9104'
    - '10.11.7.55:9104'
    - '10.11.7.71:9104'
    - '10.11.7.87:9104'

# 写一起
- labels:
    desc: mysql     # 无实际意义
  targets: ['10.11.7.29:9104','10.11.7.34:9104','10.11.7.55:9104','10.11.7.71:9104','10.11.7.87:9104']


# json模式  file_sd_configs中同步修改
 vi /usr/local/prometheus/device/mysql_device.json
[{
	"labels": {
		"desc": "mysql"
	},
	"targets": [
		"10.11.7.29:9104",
		"10.11.7.34:9104",
		"10.11.7.55:9104",
		"10.11.7.71:9104",
		"10.11.7.87:9104"
	]
}]

······
# 重启Prometheus服务，并访问prometheus所在的target状态，如果为UP说明job设置成功：
[root@prometheus ~]# cd /usr/local/prometheus
[root@prometheus prometheus]# ./promtool check config prometheus.yml
# 保持热加载
[root@prometheus ~]# curl  -X POST localhost:9090/-/reload
```
![](https://qiufuqi.github.io/img/hexo/20231207154113.png)



# Grafana中展示
在 Grafana 中导入 7362 模板 [grafana模板大全参考](https://grafana.com/grafana/dashboards/)
![](https://qiufuqi.github.io/img/hexo/20231205171035.png)
![](https://qiufuqi.github.io/img/hexo/20231207154323.png)
![](https://qiufuqi.github.io/img/hexo/20231207161206.png)

# 编写Mysql告警规则
### 编写告警规则 参考：Mysql-alert-rules.yml
建议原样拷贝，格式很重要，可以通过命令行检测
/usr/local/prometheus/promtool check config prometheus.yml
```bash
groups:
  - name: Mysql-alert
    rules:
    - alert: MySQL is down
      expr: mysql_up == 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Instance {{ $labels.instance }} MySQL is down"
        description: "MySQL database is down. This requires immediate action!"  
        resolvetion: "database has recovered."          
    - alert: open files high
      expr: mysql_global_status_innodb_num_open_files{job=~"Mysql"} > (mysql_global_variables_open_files_limit{job=~"Mysql"}) * 0.75
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Instance {{ $labels.instance }} open files high"
        description: "Open files is high. Please consider increasing open_files_limit."   
        resolvetion: "Open files has recovered."
    - alert: Read buffer size is bigger than max. allowed packet size
      expr: mysql_global_variables_read_buffer_size{job=~"Mysql"} > mysql_global_variables_slave_max_allowed_packet{job=~"Mysql"} 
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Instance {{ $labels.instance }} Read buffer size is bigger than max. allowed packet size"
        description: "Read buffer size (read_buffer_size) is bigger than max. allowed packet size (max_allowed_packet).This can break your replication."
        resolvetion: "Read buffer size has recovered."
    - alert: Sort buffer possibly missconfigured
      expr: mysql_global_variables_innodb_sort_buffer_size{job=~"Mysql"} <256*1024 or mysql_global_variables_read_buffer_size{job=~"Mysql"} > 4*1024*1024 
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Instance {{ $labels.instance }} Sort buffer possibly missconfigured"
        description: "Sort buffer size is either too big or too small. A good value for sort_buffer_size is between 256k and 4M."
        resolvetion: "Sort buffer size has recovered，the sort_buffer_size is between 256k and 4M."
    - alert: Thread stack size is too small
      expr: mysql_global_variables_thread_stack{job=~"Mysql"} <196608
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Instance {{ $labels.instance }} Thread stack size is too small"
        description: "Thread stack size is too small. This can cause problems when you use Stored Language constructs for example. A typical is 256k for thread_stack_size."
        resolvetion: "Thread stack size has resized."
    - alert: Used more than 80% of max connections limited 
      expr: mysql_global_status_max_used_connections{job=~"Mysql"} > mysql_global_variables_max_connections{job=~"Mysql"} * 0.8
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Instance {{ $labels.instance }} Used more than 80% of max connections limited"
        description: "Connections used more than 80% of max connections limited"
        resolvetion: "Connections used has recovered."
    - alert: InnoDB Force Recovery is enabled
      expr: mysql_global_variables_innodb_force_recovery{job=~"Mysql"} != 0 
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Instance {{ $labels.instance }} InnoDB Force Recovery is enabled"
        description: "InnoDB Force Recovery is enabled. This mode should be used for data recovery purposes only. It prohibits writing to the data."
        resolvetion: "InnoDB Force Recovery has been disabled."
    - alert: InnoDB Log File size is too small
      expr: mysql_global_variables_innodb_log_file_size{job=~"Mysql"} < 16777216 
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Instance {{ $labels.instance }} InnoDB Log File size is too small"
        description: "The InnoDB Log File size is possibly too small. Choosing a small InnoDB Log File size can have significant performance impacts."
        resolvetion: "The InnoDB Log File size is more than 16777216."
    - alert: InnoDB Flush Log at Transaction Commit
      expr: mysql_global_variables_innodb_flush_log_at_trx_commit{job=~"Mysql"} != 1
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Instance {{ $labels.instance }} InnoDB Flush Log at Transaction Commit"
        description: "InnoDB Flush Log at Transaction Commit is set to a values != 1. This can lead to a loss of commited transactions in case of a power failure."
        resolvetion: "InnoDB Flush Log at Transaction Commit is set to a values == 1."
    - alert: Table definition cache too small
      expr: mysql_global_status_open_table_definitions{job=~"Mysql"} > mysql_global_variables_table_definition_cache{job=~"Mysql"}
      for: 1m
      labels:
        severity: page
      annotations:
        summary: "Instance {{ $labels.instance }} Table definition cache too small"
        description: "Your Table Definition Cache is possibly too small. If it is much too small this can have significant performance impacts!"
        resolvetion: "Your Table Definition Cache is OK."
    - alert: Table open cache too small
      expr: mysql_global_status_open_tables{job=~"Mysql"} >mysql_global_variables_table_open_cache{job=~"Mysql"} * 99/100
      for: 1m
      labels:
        severity: page
      annotations:
        summary: "Instance {{ $labels.instance }} Table open cache too small"
        description: "Your Table Open Cache is possibly too small (old name Table Cache). If it is much too small this can have significant performance impacts!"
        resolvetion: "Your Table Open Cache is OK."
    - alert: Thread stack size is possibly too small
      expr: mysql_global_variables_thread_stack{job=~"Mysql"} < 262144
      for: 1m
      labels:
        severity: page
      annotations:
        summary: "Instance { $labels.instance }} Thread stack size is possibly too small"
        description: "Thread stack size is possibly too small. This can cause problems when you use Stored Language constructs for example. A typical is 256k for thread_stack_size."
        resolvetion: "Thread stack size is OK."
    # - alert: InnoDB Buffer Pool Instances is too small
        # expr: mysql_global_variables_innodb_buffer_pool_instances == 1
        # for: 1m
        # labels:
          # severity: page
        # annotations:
          # summary: "Instance {{ $labels.instance }} InnoDB Buffer Pool Instances is too small"
          # description: "If you are using MySQL 5.5 and higher you should use several InnoDB Buffer Pool Instances for performance reasons. Some rules are: InnoDB Buffer Pool Instance should be at least 1 Gbyte in size. InnoDB Buffer Pool Instances you can set equal to the number of cores of your machine."
    - alert: InnoDB Plugin is enabled
      expr: mysql_global_variables_ignore_builtin_innodb{job=~"Mysql"} == 1
      for: 1m
      labels:
        severity: page
      annotations:
        summary: "Instance {{ $labels.instance }} InnoDB Plugin is enabled"
        description: "InnoDB Plugin is enabled"
        resolvetion: "InnoDB Plugin is disabled."
    - alert: Binary Log is disabled
      expr: mysql_global_variables_log_bin{job=~"Mysql"} != 1
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Instance { $labels.instance }} Binary Log is disabled"
        description: "Binary Log is disabled. This prohibits you to do Point in Time Recovery (PiTR)."
        resolvetion: "Binary Log is enabled."
    # - alert: Binlog Cache size too small
        # expr: mysql_global_variables_binlog_cache_size < 1048576
        # for: 1m
        # labels:
          # severity: page
        # annotations:
          # summary: "Instance {{ $labels.instance }} Binlog Cache size too small"
          # description: "Binlog Cache size is possibly to small. A value of 1 Mbyte or higher is OK."
         
    # - alert: Binlog Statement Cache size too small
        # expr: mysql_global_variables_binlog_stmt_cache_size <1048576 and mysql_global_variables_binlog_stmt_cache_size > 0
        # for: 1m
        # labels:
          # severity: page
        # annotations:
          # summary: "Instance {{ $labels.instance }} Binlog Statement Cache size too small"
          # description: "Binlog Statement Cache size is possibly to small. A value of 1 Mbyte or higher is typically OK."
    # - alert: Binlog Transaction Cache size too small
        # expr: mysql_global_variables_binlog_cache_size  <1048576
        # for: 1m
        # labels:
          # severity: page
        # annotations:
          # summary: "Instance {{ $labels.instance }} Binlog Transaction Cache size too small"
          # description: "Binlog Transaction Cache size is possibly to small. A value of 1 Mbyte or higher is typically OK."
    # - alert: Sync Binlog is enabled
        # expr: mysql_global_variables_sync_binlog == 1
        # for: 1m
        # labels:
          # severity: page
        # annotations:
          # summary: "Instance {{ $labels.instance }} Sync Binlog is enabled"
          # description: "Sync Binlog is enabled. This leads to higher data security but on the cost of write performance."
    # - alert: IO thread stopped
        # expr: mysql_slave_status_slave_io_running != 1
        # for: 1m
        # labels:
          # severity: critical
        # annotations:
          # summary: "Instance {{ $labels.instance }} IO thread stopped"
          # description: "IO thread has stopped. This is usually because it cannot connect to the Master any more."
    # - alert: SQL thread stopped 
        # expr: mysql_slave_status_slave_sql_running == 0
        # for: 1m
        # labels:
          # severity: critical
        # annotations:
          # summary: "Instance {{ $labels.instance }} SQL thread stopped"
          # description: "SQL thread has stopped. This is usually because it cannot apply a SQL statement received from the master."
    # - alert: SQL thread stopped
        # expr: mysql_slave_status_slave_sql_running != 1
        # for: 1m
        # labels:
          # severity: critical
        # annotations:
          # summary: "Instance {{ $labels.instance }} Sync Binlog is enabled"
          # description: "SQL thread has stopped. This is usually because it cannot apply a SQL statement received from the master."
    # - alert: Slave lagging behind Master
        # expr: rate(mysql_slave_status_seconds_behind_master[1m]) >30 
        # for: 1m
        # labels:
          # severity: warning 
        # annotations:
          # summary: "Instance {{ $labels.instance }} Slave lagging behind Master"
          # description: "Slave is lagging behind Master. Please check if Slave threads are running and if there are some performance issues!"
    # - alert: Slave is NOT read only(Please ignore this warning indicator.)
        # expr: mysql_global_variables_read_only != 0
        # for: 1m
        # labels:
          # severity: page
        # annotations:
          # summary: "Instance {{ $labels.instance }} Slave is NOT read only"
          # description: "Slave is NOT set to read only. You can accidentally manipulate data on the slave and get inconsistencies..."
 
```
