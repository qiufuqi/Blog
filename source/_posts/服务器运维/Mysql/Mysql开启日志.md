---
title: Mysql日志
date: 2022-09-7
tags:
  - Linux
  - Mysql
  - 日志
categories: 
- 运维
- 数据库
- Mysql
- 日志
keywords: 'Linux,Mysql,日志'
cover: https://qiufuqi.github.io/img/hexo/20220922143426.png
abbrlink: mysql5.7_RZ
comments: false
---

``` bash
1.进入MySQL，开启日志选项（默认情况应该是关闭的）：
mysql> set global general_log=on;

2. 查询本机MySQL执行日志保存的路径：
mysql> show variables like 'general_log_file';

3. 重启MySQL服务器：
systemctl restart mysqld

4. 查看MySQL执行日志：

# 路径也就是在第2步中查询出的
tail -f /path/to/general_log_file


# 日志清理
show binary logs;
PURGE BINARY LOGS TO  'mysql-bin.033662';
```