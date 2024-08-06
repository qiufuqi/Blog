---
title: Linux实用脚本
date: 2023-12-08
tags:
  - Linux
  - CentOS
  - 实用脚本
categories: 
- 运维
- 实用脚本
keywords: 'Linux,CentOS,实用脚本'
cover: https://qiufuqi.github.io/img/hexo/20230418114913.png
abbrlink: centos_shell
comments: false
---

**Linux实用脚本**

## 自动实时同步数据
使用INOTIFY+RSYNC自动实时同步数据  inotify_rsyncs.sh
```bash
#!/bin/bash
# Author: Austines
# chkconfig: - 85 15
# description: It is used to serve
# 监测/data路径下的文件变化，排除Temp目录
INOTIFY_CMD="inotifywait -mrq -e modify,create,move,delete /data/ --exclude=Temp"
# 同步数据
RSYNC_CMD1="rsync -avz /data/ --exclude-from=/etc/rc.d/init.d/exclude.txt harry@10.11.7.68:/data/ --delete"
RSYNC_CMD2="rsync -avz /data/ --exclude-from=/etc/rc.d/init.d/exclude.txt harry@10.11.7.68:/data/ --delete"
$INOTIFY_CMD | while read DIRECTORY EVENT FILE
do
    if [ $(pgrep rsync | wc -l) -le 0 ] ; then
        $RSYNC_CMD1&&$RSYNC_CMD2 >> rsync.log
    fi
done
```

## MYSQL自动备份
MYSQL自动备份以及删除备份脚本 db_backup.sh
```bash
#!/bin/bash
# Author: Austines
# Description: Database backup script
dbback(){
# 定义变量
db_user="ma_prd"
db_passwd="<password>"
db_path="/data/bakmysql"
db_file="backuprecord"
db_date=`date +%Y%m%d_%H:%M:%S`
# 判断路径是否存在
[ -d $db_path ] || exit 2
# 使用mysqldump备份数据，并用gzip进行压缩
mysqldump -u$db_user  -p$db_passwd --single-transaction ma  | gzip > $db_path/${db_date}_ma.sql.gz
REVAL=$?
if [ $REVAL -eq 0 ]
    then
        echo "$db_date ma db is backups successful" >>$db_path/$db_file
    else
        echo "$db_date ma db is backups failed" >>$db_path/$db_file
fi
}

#删除超过7天的备份数据
delbak(){
local db_path="/data/bakmysql"
find $db_path -type f -name "*ma*.gz" -mtime +7 -exec rm -rf {} \;
}
dbback
delbak
```

## 文件自动备份
```bash
#!/bin/bash
# Author: Austines
# Description: files backup script

# 备份源文件路径
source_files=(
    "/usr/local/nginx/conf/nginx.conf"
    "/usr/local/nginx/conf/tcp.stream"
    "/usr/local/src/check_nginx_pid.sh"
    "/home/wwwroot/default/it/"
)

backup_dir="/home/backup"       # 备份目标路径
retention_days=90               # 保留天数
mkdir -p $backup_dir            # 创建备份目录（如果不存在）

# 备份文件
for source_file in "${source_files[@]}"; do
    backup_file=$(basename $source_file).$(date +%Y%m%d%H%M%S)
    cp -r $source_file $backup_dir/$backup_file
    echo "备份成功: $backup_dir/$backup_file"
done

# 删除过期的备份文件
find $backup_dir -type f -mtime +$retention_days -exec rm {} \;
# 打印备份成功消息
echo "备份完成"

```

## 检测网站可用性
使用curl检测网站可用性脚本 web_check_with_curl.sh
```bash
#!/bin/bash
# Author: Austines
# Version：1.1
# Description: Web check with curl

#定义颜色
red='\e[0;31m'
RED='\e[1;31m'
green='\e[0;32m'
GREEN='\e[1;32m'
blue='\e[0;34m'
BLUE='\e[1;34m'
cyan='\e[0;36m'
CYAN='\e[1;36m'
NC='\e[0m'
date=`date +%Y-%m-%d' '%H:%M:%S` 
# 定义User Agent
ua="Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/62.0.3202.9 Safari/537.36"
pass_count=0
fail_count=0
# 需要检测的url
urls=(
    "http://www.xxx.com"
)

function request(){
    status=$(curl -sk -o /dev/null --retry 1 --connect-timeout 1 -w '%{http_code}' --user-agent "$ua" $1)
    if [ $status -eq '200' -o $status -eq '301' \
                           -o $status -eq '302' ]; then
        echo -e "[${GREEN} Passed ${NC}] => $1"
  ((pass_count ++))
    else
        echo -e "[${RED} Failed ${NC}] => $1"
  ((fail_count ++))
    fi
}

function main(){
    echo "Start checking ..."
    for((i=0;i<${#urls[*]};i++)) 
        do 
        request ${urls[i]};
        done
    # 输出检测通过和失败的记录
    echo -e "======================== Summary ======================== "
    echo -e "Total: ${cyan} $((pass_count + fail_count))${NC}  Passed: ${green}${pass_count}${NC}  Failed: ${red}${fail_count}${NC} Time: $date"
       
}

main $*
```

## 检测软件是否运行
利用脚本定时监控本地、远端数据库服务端或Web服务是否运行正常，例如：负载高、cup高、连接数满了。[参考](https://www.cnblogs.com/su-root/p/10409973.html)
建议通过专业的监控软件实施监控，比如Prometheus，zabbix等。
``` bash
## 本地端口监测     netstat/ss/lsof
netstat -lntup|grep 3306|wc -l
ss -lntup|grep 3306|wc -l
lsof -i:3306|wc -l

### 远程端口监测    telnet/nmap/nc
echo -e "\n"|telnet baidu.com 80|grep Connected|wc
namp www.baidu.com -p 80|grep open|wc -l


#!/bin/sh
#if [ `lsof -i tcp:3306|wc -l` -gt 0 ]
#if [ `ps -ef|grep mysql|grep -v grep|wc -l` -gt 0 ] #注意脚本名字不能带mysql，自己也算进程
#if [ `netstat -lntup|grep mysql|wc -l` -gt 0 ] 
if [ "`netstat -lnt|grep 3306|awk -F "[ :]+" '{print $4}'`" = "3306" ]
      then
    echo "MySQL is Running."
else
   echo "MySQL is Stopped."
   systemctl start mysqld
   echo "MySQL is Starting......"
fi


# 如果mysql没启动，空值-eq 3306 会报错，如果以字符串的方式比较不会。
[root@devqiu ~]# sh my.sh 
MySQL is Stopped.
MySQL is Starting......
[root@devqiu ~]# sh my.sh 
MySQL is Running.
```

## 封禁异常IP地址
检测并封禁异常IP地址的脚本
```bash
#!/bin/bash

# 获取当前日期和时间的格式化字符串
DATE=$(date +%d/%b/%Y:%H:%M)

# 日志文件路径和封禁记录文件路径
LOG_FILE="/usr/local/nginx/logs/access.log"
BANNED_IP_LOG="/usr/local/nginx/logs/banned_ip.log"

# 获取异常IP地址，使用tail命令读取日志文件的最后10000行，并使用grep命令筛选出包含当前日期和时间的日志记录
ABNORMAL_IP=$(tail -n 10000 "$LOG_FILE" | grep "$DATE" | awk '{a[$1]++}END{for(i in a) if(a[i]>10) print i}')

# 封禁异常IP地址
declare -a IP_LIST
for IP in $ABNORMAL_IP; do
    if ! iptables -vnL | grep -q "$IP"; then
        iptables -I INPUT -s "$IP" -j DROP
        echo "$(date +'%F_%T') $IP" >> "$BANNED_IP_LOG"
        IP_LIST+=("$IP")
    fi
done

# 打印被封禁的IP地址
if [ ${#IP_LIST[@]} -gt 0 ]; then
    echo "以下IP地址已被封禁："
    printf "%s\n" "${IP_LIST[@]}"
else
    echo "没有需要封禁的IP地址。"
fi
```

## 网卡实时流量
查看网卡实时流量脚本 bash interface_moniter.sh eth0
```bash
#!/bin/bash
# 如果没有传递参数，默认使用 lo 作为网络接口
NIC=${1:-lo}
echo -e " In ------ Out"
while true; do
    # 使用awk命令从/proc/net/dev文件中提取指定网络接口的接收字节数和发送字节数，并保存到变量OLD_IN和OLD_OUT中
    OLD_IN=$(awk  '$0~"'$NIC'"{print $2}' /proc/net/dev)
    OLD_OUT=$(awk '$0~"'$NIC'"{print $10}' /proc/net/dev)
    # 等待1秒钟
    sleep 1
    # 再次使用awk命令提取最新的接收字节数和发送字节数，并保存到变量NEW_IN和NEW_OUT中。
    NEW_IN=$(awk  '$0~"'$NIC'"{print $2}' /proc/net/dev)
    NEW_OUT=$(awk '$0~"'$NIC'"{print $10}' /proc/net/dev)
    # 计算接收速率和发送速率，单位为KB/s，并保存到变量IN和OUT中
    IN=$(printf "%.1f%s" "$((($NEW_IN-$OLD_IN)/1024))" "KB/s")
    OUT=$(printf "%.1f%s" "$((($NEW_OUT-$OLD_OUT)/1024))" "KB/s")
    # 使用echo命令输出接收速率和发送速率
    echo "$IN $OUT"
    sleep 1
done
```

## 日志分析脚本
访问日志分析脚本 bash log_analyze.sh access.log
```bash
#!/bin/bash
# 日志格式: $remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" "$http_x_forwarded_for"
LOG_FILE=$1

echo "统计访问最多的10个IP"
awk '{a[$1]++}END{print "UV:",length(a);for(v in a)print v,a[v]}' $LOG_FILE | sort -k2 -nr | head -10
echo "----------------------"

echo "统计时间段访问最多的IP"
awk '$4>="[01/Dec/2018:13:20:25" && $4<="[27/Nov/2018:16:20:49"{a[$1]++}END{for(v in a)print v,a[v]}' $LOG_FILE | sort -k2 -nr | head -10
echo "----------------------"

echo "统计访问最多的10个页面"
awk '{a[$7]++}END{print "PV:",length(a);for(v in a){if(a[v]>10)print v,a[v]}}' $LOG_FILE | sort -k2 -nr
echo "----------------------"

echo "统计访问页面状态码数量"
awk '{a[$7" "$9]++}END{for(v in a){if(a[v]>5)print v,a[v]}}' $LOG_FILE
```

## 服务器运行分析
Linux实时信息&状态监控脚本
```bash
#!/bin/bash
# 获取服务器基本信息
hostname=$(hostname)
ip_address=$(hostname -I | awk '{print $1}')
os=$(lsb_release -ds)
kernel=$(uname -r)
uptime=$(uptime -p)
# 监控循环
while true; do
    # 获取CPU信息
    cpu_model=$(cat /proc/cpuinfo | grep "model name" | head -n1 | awk -F': ' '{print $2}')
    cpu_cores=$(cat /proc/cpuinfo | grep "model name" | wc -l)
    # 获取内存信息（加入单位）
    memory_total=$(free -h | awk 'NR==2{print $2}')
    memory_used=$(free -h | awk 'NR==2{print $3}')
    memory_free=$(free -h | awk 'NR==2{print $4}')
    memory_available=$(free -h | awk 'NR==2{print $7}')
    # 获取磁盘使用情况
    disk_total=$(df -h --output=size / | awk 'NR==2{print $1}')
    disk_used=$(df -h --output=used / | awk 'NR==2{print $1}')
    disk_free=$(df -h --output=avail / | awk 'NR==2{print $1}')
    # 使用 top 命令获取 CPU 使用率
    cpu_usage=$(top -b -n 1 | grep "%Cpu(s):" | awk '{printf "%.2f%%", 100-$8}')
    # 输出监控信息
    clear
    echo "服务器信息："
    echo "主机名：$hostname"
    echo "IP地址：$ip_address"
    echo "操作系统：$os"
    echo "内核版本：$kernel"
    echo "运行时间：$uptime"
    echo "--------------------------------------"
    echo "CPU信息："
    echo "型号：$cpu_model"
    echo "核心数：$cpu_cores"
    echo "CPU使用率：$cpu_usage"
    echo "--------------------------------------"
    echo "内存信息："
    echo "总量：$memory_total"
    echo "已使用：$memory_used"
    echo "可用：$memory_available"
    echo "--------------------------------------"
    echo "磁盘信息："
    echo "总量：$disk_total"
    echo "已使用：$disk_used"
    echo "可用：$disk_free"

    # 每 3 秒刷新一次
    sleep 3
done
```
运行结果
![](https://qiufuqi.github.io/img/hexo/20231208143947.png)

