---
title: Mysql备份脚本
date: 2023-12-08
tags:
  - Linux
  - CentOS
  - Mysql
categories: 
- 运维
- 数据库
- Mysql
- 备份
keywords: 'Linux,CentOS,Mysql,备份'
description: Mysql备份脚本
cover: https://qiufuqi.github.io/img/hexo/20220909115022.png
abbrlink: centos_mysql_backup
comments: false
---

Mysql数据库备份脚本

## 库备份
备份mysql数据库，除了一些系统库，其他全部备份，备份目录可自己指定。
```bash
#!/bin/bash
DATE=$(date +%F_%H-%M-%S)
HOST=localhost
USER=backup
PASS=123.com
BACKUP_DIR=/home/db_backup
# 保留天数
retention_days=30

DB_LIST=$(mysql -h$HOST -u$USER -p$PASS -s -e "show databases;" 2>/dev/null |egrep -v "Database|information_schema|mysql|performance_schema|sys")
 
for DB in $DB_LIST; do
    BACKUP_NAME=$BACKUP_DIR/${DB}_${DATE}.sql
    if ! mysqldump -h$HOST -u$USER -p$PASS -B $DB > $BACKUP_NAME 2>/dev/null; then
        echo "$BACKUP_NAME 备份失败!"
    fi
done

find $BACKUP_DIR -type f -mtime +$retention_days -exec rm {} \;
echo "备份结束"
```

## 表备份
备份mysql数据库的每张表，每张表单独sql。
```bash
#!/bin/bash
DATE=$(date +%F_%H-%M-%S)
HOST=localhost
USER=backup
PASS=123.com
BACKUP_DIR=/home/db_backup
# 保留天数
retention_days=30

DB_LIST=$(mysql -h$HOST -u$USER -p$PASS -s -e "show databases;" 2>/dev/null |egrep -v "Database|information_schema|mysql|performance_schema|sys")
 
for DB in $DB_LIST; do
    BACKUP_DB_DIR=$BACKUP_DIR/${DB}_${DATE}
    [ ! -d $BACKUP_DB_DIR ] && mkdir -p $BACKUP_DB_DIR &>/dev/null
    TABLE_LIST=$(mysql -h$HOST -u$USER -p$PASS -s -e "use $DB;show tables;" 2>/dev/null)
    for TABLE in $TABLE_LIST; do
        BACKUP_NAME=$BACKUP_DB_DIR/${TABLE}.sql
        if ! mysqldump -h$HOST -u$USER -p$PASS $DB $TABLE > $BACKUP_NAME 2>/dev/null; then
            echo "$BACKUP_NAME 备份失败!"
        fi
    done
done

find $BACKUP_DIR -type f -mtime +$retention_days -exec rm {} \;
echo "备份结束"
```

## 异地备份
使用mkdir -p命令创建以当前日期为名的目录，存放数据库备份文件。
使用mysqldump命令备份所有数据库，并将输出重定向至mysql_backup_$DATE.sql文件中。
使用tar命令将备份文件压缩为：.tar.gz格式的文件内，并附加日志到备份文件中，然后删除原始的备份文件。
使用rm -rf命令删除 7 天前的备份文件，删除的是以 7 天前日期为名的目录和目录下的所有文件。
使用scp命令将本地备份文件传到远程备份服务器上。

远程备份需要做免密，在MySQL数据库所在服务器与需要远程备份的服务器做免密，并进行ssh验证登录是否正常。
```bash
ssh-keygen
cd ~/.ssh/
scp id_rsa.pub 目标主机IP：~/.ssh/authorized_keys
```
执行脚本
```bash
#!/bin/bash
# Database info
DB_USER="root"                          # 数据库备份用户
DB_PASS="1Q!2W@3E#"                     # 备份用户密码
DB_HOST="192.168.1.100"                 # 数据库 IP
DBBACK_IP="192.168.1.200"               # 远程备份服务器 IP
BCK_DIR="/bigdata/mysql"                # 本地备份路径
DBBACK_PATH=/bigdata/mysqlbackup        # 远程备份路径
DATE=`date +%F`                         # 获取当前时间
yestoday=$(date -d '-7 day' +%Y-%m-%d)  # 取 7 天前的时间，格式为：2023-12-30，用于删除备份文件取文件时间，该参数可自行调整

#BACK_NAME="db_$var.sql"
#TB_NAME=("" "" "" "" "")               # 需要备份的表名

#create file

mkdir -p $BCK_DIR/$DATE                 # 创建本地备份日期目录
echo "开始本地备份中..."

mysqldump -u$DB_USER -p$DB_PASS -h$DB_HOST --all-databases > $BCK_DIR/$DATE/mysql_backup_$DATE.sql
cd $BCK_DIR/$DATE && tar -zcvf mysql_backup_$DATE.sql.tar.gz mysql_backup_$DATE.sql >>/$BCK_DIR/$DATE/$DATE.log && rm -fr mysql_backup_$DATE.sql
echo "$DATE db bakcup success！" >>/$BCK_DIR/$DATE/$DATE.log

echo "开始删除 7 天前的数据库备份文件..."
rm -rf $BCK_DIR/$yestoday
echo "7 天前的数据库备份文件删除完毕！"

echo "开始远程备份中..."
scp -r $BCK_DIR/$DATE/mysql_backup_$DATE.sql.tar.gz root@$DBBACK_IP:$DBBACK_PATH
echo "远程备份完毕！"
```