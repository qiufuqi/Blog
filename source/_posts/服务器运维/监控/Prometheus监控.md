---
title: Prometheus监控+altermanager+钉钉告警
date: 2022-08-13 15:45:50
tags:
  - Prometheus
  - Altermanager
  - Grafana
  - 钉钉告警
categories: 
- 运维
- 监控
keywords: 'Prometheus,监控,钉钉告警,altermanager'
description: Prometheus监控安装,告警
cover: https://qiufuqi.github.io/img/hexo/20231205135025.png
abbrlink: prometheus
url: prometheus
comments: false
top: 1000
---

Prometheus 是一个开源的服务监控系统和时序数据库，其提供了通用的数据模型和快捷数据采集、存储和查询接口。它的核心组件Prometheus server会定期从静态配置的监控目标或者基于服务发现自动配置的自标中进行拉取数据，当新拉取到的数据大于配置的内存缓存区时，数据就会持久化到存储设备当中。
Prometheus 的数据走向，如下：
![](https://qiufuqi.github.io/img/hexo/20231205114030.png)

| Align `left`   | center align |   Align right |
| :------------- | :----------: | ------------: |
| `left`-aligned |   centered   | right-aligned |
| `左`对齐        |    中对齐     |         右对齐 |

| 对象类型   | 具体对象 | 采集器 |
| :------------- | :---------- | :---------- |
| 网络协议 | HTTP、HTTPS、DNS、TCP、ICMP 和 gRPC等| Blackbox Exporter |
| 网络设备 | 路由器、交换机等| SNMP Exporter |
| 主机节点 | 虚拟主机、物理主机等 | Node Exporter、Windows Exporter |
| 应用 | 延迟、错误、QPS、内部状态等 | Prometheus Client libraries |
| 中间件 | 资源用量、服务状态等 | Prometheus Client libraries |
| 容器 | 资源用量、状态等 | cAdvisor |
| 编排工具 | 集群资源用量、调度等 | Kubernetes Components |
| 数据库 | 常用数据库：MySQL、Redis、MongoDB、SQL Server | MySQL Exporter, Redis Exporter, MongoDB Exporter, MSSQL Exporter |
| 硬件 | APC UPS，IoT，物理服务器状态IPMI | Apcupsd Exporter，IoT Edison Exporter， IPMI Exporter |
| 消息队列 | 常见的消息队列中间件 | Beanstalkd Exporter，Kafka Exporter, NSQ Exporter, RabbitMQ Exporter |
| 存储 | Ceph、Gluster、HDFS、ScaleIO | Ceph Exporter, Gluster Exporter, HDFS Exporter, ScaleIO Exporter |
| HTTP服务 | Apache、Nginx、HAProxy | Apache Exporter, HAProxy Exporter, Nginx Exporter |
| API服务 | AWS ECS、Docker Cloud、Docker Hub、GitHub | AWS ECS Exporter， Docker Cloud Exporter, Docker Hub Exporter, GitHub Exporter |
| 日志 | Fluentd、Grok | Fluentd Exporter, Grok Exporter |
| 监控系统 | Graphite、InfluxDB、Nagios、Collectd | Collectd Exporter, Graphite Exporter, InfluxDB Exporter, Nagios Exporter |
| 其它 | JIRA、Jenkins、Confluence | JIRA Exporter, Jenkins Exporter， Confluence Exporter |

## 系统环境准备
准备一台centos7.6系统，ip：10.11.7.95（根据自己实际情况）

### 关闭selinux
```
vi /etc/sysconfig/selinux
    SELINUX=disabled
setenforce 0
```
### 关闭防火墙或者开放对应端口
所需端口：prometheus/9090/tcp grafana/3000/tcp alertmanager/9093/tcp prometheus-ding/8060/tcp

```  bash
systemctl stop firewalld
systemctl start firewalld

# 开放端口
firewall-cmd --zone=public --list-ports 查看端口
firewall-cmd --zone=public --query-port=80/tcp    查看端口
firewall-cmd --zone=public --add-port=3306/tcp --permanent        添加端口
firewall-cmd --zone=public --remove-port=80/tcp --permanent    删除端口
firewall-cmd --reload
systemctl reload firewalld.service
```

## 安装Prometheus平台
从[官网1](https://prometheus.io/download/),[官网2](https://github.com/orgs/prometheus/repositories?type=all)下载相应版本并安装，数据缓存在data目录中
``` bash
cd /home/ && mkdir package && cd package
 
# 下载对应安装包：
wget https://github.com/prometheus/prometheus/releases/download/v2.37.0/prometheus-2.37.0.linux-amd64.tar.gz
 
# 解压至指定文件夹
tar -zxvf /home/package/prometheus-2.37.0.linux-amd64.tar.gz -C /usr/local/
 
# 创建软链
ln -s /usr/local/prometheus-2.37.0.linux-amd64/ /usr/local/prometheus
 
# 配置prometheus开机自启
vi /usr/lib/systemd/system/prometheus.service
# 写入以下信息：
[Unit]
Description=https://prometheus.io
 
[Service]
Restart=on-failure
ExecStart=/usr/local/prometheus/prometheus --config.file=/usr/local/prometheus/prometheus.yml --storage.tsdb.path=/usr/local/prometheus/data --web.enable-lifecycle
[Install]
WantedBy=multi-user.target
 
systemctl daemon-reload
systemctl start prometheus
systemctl status prometheus
systemctl enable prometheus
```
访问地址：http://10.11.7.95:9090/ 可查看对应的数据源
### 热启动
prometheus启动后修改配置文件就需要再重启生效，可以通过以下方式 热加载
```bash
curl -X POST http://localhost:9090/-/reload
```
请求接口后返回 Lifecycle API is not enabled. 那么就是启动的时候没有开启热更新配置，需要在启动的命令行增加参数： --web.enable-lifecycle
```bash
vi /usr/lib/systemd/system/prometheus.service
···
ExecStart=/usr/local/prometheus/prometheus --config.file=/usr/local/prometheus/prometheus.yml --storage.tsdb.path=/usr/local/prometheus/data --web.enable-lifecycle
···
# 然后执行命令
systemctl daemon-reload
systemctl restart prometheus

#后面每次修改了prometheus配置文件后，可以调用接口进行配置的热加载：
curl -X POST http://localhost:9090/-/reload
```


## 安装windows_exporter
从[官网](https://github.com/prometheus-community/windows_exporter/releases)下载相应版本并安装
下载msi版本，双击安装，会开机自动启动
![](https://qiufuqi.github.io/img/hexo/20221012103257.png)
访问地址：http://目标ip:9182/metrics 可查看对应的数据源

## 安装node_exporter 
从[官网](https://prometheus.io/download/)下载相应版本并安装

### 在各个节点服务器上安装监控
安装node_exporter，采集机器运行数据信息，默认端口9100 （可更改为指定端口）,按照步骤执行以下命令或者拷贝脚本运行
#### 脚本安装node
node_exporter.sh一键监控安装脚本，提前下载好安装文件或者[在线下载](https://github.com/prometheus/blackbox_exporter/releases)
```bash
#!/bin/bash
# -*- coding: utf-8 -*-
# Date: 2023/12/13
echo "download node_exporter"
sleep 2
wget -N -P /root/ https://github.com/prometheus/node_exporter/releases/download/v1.4.0-rc.0/node_exporter-1.4.0-rc.0.linux-amd64.tar.gz

echo "tar node_exporter"
sleep 2
tar -zxvf /root/node_exporter-1.4.0-rc.0.linux-amd64.tar.gz -C /opt/ && mv /opt/node_exporter-1.4.0-rc.0.linux-amd64 /usr/local/node_exporter

echo "delete node_exporter***tar.gz"
rm -rf /root/node_exporter-1.4.0-rc.0.linux-amd64.tar.gz


echo "firewall node_exporter port 9100 "
sleep 2
firewall-cmd --zone=public --add-port=9100/tcp --permanent && firewall-cmd --reload 
 
echo "add node_exporter.service"
sleep 2
cat << EOF > /usr/lib/systemd/system/node_exporter.service
[Unit]
Description=node_exporter
After=network.target 
 
[Service]
Restart=on-failure
ExecStart=/usr/local/node_exporter/node_exporter --web.listen-address=:9100

[Install]
WantedBy=multi-user.target
EOF
 
echo "start node_exporter.service"
sleep 2
systemctl daemon-reload && systemctl start node_exporter && systemctl enable --now node_exporter
```

#### 逐步安装node
``` bash
cd /home/ && mkdir package && cd package
 
下载对应安装包：
wget https://github.com/prometheus/node_exporter/releases/download/v1.4.0-rc.0/node_exporter-1.4.0-rc.0.linux-amd64.tar.gz
 
解压至指定文件夹
tar -zxvf /home/package/node_exporter-1.4.0-rc.0.linux-amd64.tar.gz -C /usr/local/
 
创建软链
ln -s /usr/local/node_exporter-1.4.0-rc.0.linux-amd64/ /usr/local/node_exporter
 
配置node_exporter开机自启
vi /usr/lib/systemd/system/node_exporter.service
写入以下信息：
[Unit]
Description=node_exporter
After=network.target 
 
[Service]
Restart=on-failure
ExecStart=/usr/local/node_exporter/node_exporter --web.listen-address=:9100

[Install]
WantedBy=multi-user.target

#重启服务
systemctl daemon-reload
systemctl start node_exporter
systemctl status node_exporter
systemctl enable node_exporter
```
访问地址：http://目标ip:9100/metrics 可查看对应的数据源

## 加入prometheus监控
### 静态部署
使用static_configs实现静态部署，当有新增节点时需要重启
``` bash
cd /usr/local/prometheus/
cp /usr/local/prometheus/prometheus.yml /usr/local/prometheus/prometheus.yml.bak
 
# 添加对应的需监控信息
vi /usr/local/prometheus/prometheus.yml

# 示例1：
- job_name: 'Doris'
    scrape_interval:     10s
    static_configs:
      - targets: ['10.12.79.40:9100','10.12.79.41:9100','10.12.79.42:9100','10.12.79.43:9100']

# 示例2： 仅不同写法
- job_name: 'Doris'
    scrape_interval:     10s
    static_configs:
      - targets:
         - 10.12.79.40:9100
         - 10.12.79.41:9100

# 检测填写是否正确
./promtool check config prometheus.yml
 Checking prometheus.yml
 SUCCESS: prometheus.yml is valid prometheus config file syntax
 
# 重启prometheus
systemctl restart prometheus
# 保持热加载
curl  -X POST localhost:9090/-/reload
```
### 文件部署
当有新的节点时，只需要修改对应的yml或者json文件即可。
``` bash
# 添加对应的需监控信息
vi /usr/local/prometheus/prometheus.yml

# 文件发现服务
- job_name: 'Doris'
  scrape_interval:     10s
  file_sd_configs: # 基于文件服务发现
    - files:
      - "./device/doris_device.yml"    # - "./device/doris_device.json"  yml和json格式选择一个即可
      refresh_interval: 5s

 vi /usr/local/prometheus/device/doris_device.yml
 # 或者单独列
- labels:
    desc: Doris     # 无实际意义
  targets:
    - '10.12.79.40:9100'
    - '10.12.79.41:9100'

# 写一起
- labels:
    desc: Doris     # 无实际意义
  targets: ['10.12.79.40:9100','10.12.79.41:9100']


# json模式  file_sd_configs中同步修改
 vi /usr/local/prometheus/device/doris_device.json
[{
	"labels": {
		"desc": "Doris"
	},
	"targets": [
		"10.12.79.40:9100",
		"10.12.79.41:9100"
	]
}]

# 检测填写是否正确
./promtool check config prometheus.yml
# 重启prometheus
systemctl restart prometheus
# 保持热加载
curl  -X POST localhost:9090/-/reload
```

访问地址：http://10.11.7.95:9090/targets 可查看是否捕捉到信息

## 搭建grafana平台
从[官网](https://grafana.com/grafana/download)下载对应版本安装
从[grafana模板](https://grafana.com/grafana/dashboards/)导入对应模板id
``` bash
wget https://dl.grafana.com/enterprise/release/grafana-enterprise-9.0.2-1.x86_64.rpm
yum -y install grafana-enterprise-9.0.2-1.x86_64.rpm
 
# 启动-并开机自启
systemctl start grafana-server
systemctl enable grafana-server
```
访问地址：http://10.11.7.95:3000/login admin/admin登陆后修改密码
Data sources添加Prometheus数据源：http://10.11.7.95:9090

导入[Linux模板](https://github.com/starsliao/Prometheus)，本次选择模板id：16098
导入Window模板，模板id：10467

## 安装alertmanager报警
### 下载对应安装包并解压
``` bash
cd /home/package
wget https://github.com/prometheus/alertmanager/releases/download/v0.21.0/alertmanager-0.21.0.linux-amd64.tar.gz
 
解压至指定文件夹
tar -zxvf /home/package/alertmanager-0.21.0.linux-amd64.tar.gz -C /usr/local/
软连接
ln -s /usr/local/alertmanager-0.21.0.linux-amd64/ /usr/local/alertmanager
```
### 配置alertmanager开机自启动
``` bash
vi /usr/lib/systemd/system/alertmanager.service

[Unit]
Description=https://prometheus.io
 
[Service]
Restart=on-failure
ExecStart=/usr/local/alertmanager/alertmanager --config.file=/usr/local/alertmanager/alertmanager.yml
[Install]
WantedBy=multi-user.target
 
启动 && 自启动
systemctl start alertmanager
systemctl enable alertmanager
 
访问地址http://10.11.7.95:9093/#/alerts
```

### 配置altermanager报警属性
``` bash
cd /usr/local/alertmanager && cp alertmanager.yml alertmanager.yml.bak
vi alertmanager.yml

#参考示例
################开始###############
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.126.com:25'      # smtp地址
  smtp_from: 'XXXXXXXX@126.com'          # 谁发邮件
  smtp_auth_username: 'XXXXXXXX@126.com' # 邮箱用户
  smtp_auth_password: 'XXXXXXXXXXXXXXXX' # 邮箱客户端授权密码
  smtp_require_tls: false
 
templates:                               # 指定邮件模板的路径，可以使用相对路径，template/*.tmp的方式
  - '/usr/local/alertmanager/template/*.tmp'
 
route:                                   # route用来设置报警的分发策略
  group_by: ["alertname"]                # 分组名
  group_wait: 30s                        # 当收到告警的时候，等待三十秒看是否还有告警，如果有就一起发出去
  group_interval: 30s                    # 发送警告间隔时间
  repeat_interval: 20m                   # 重复报警的间隔时间
  receiver: Node_warning                 # 设置默认接收人，如果想分组接收，把下面这段的注释去掉
 
receivers:                               # 定义接收者，将告警发送给谁
- name: 'Node_warning'
  email_configs:
  - send_resolved: true
    to: '******@163.com'
    html: '{{ template "email.html" . }}' # 指定使用模板，如果不指定，还是会加载默认的模板的
    headers: { Subject: "[WARN]告警" }    # 配置邮件主题
 
  webhook_configs:                        # 钉钉消息
  - url: http://127.0.0.1:8060/dingtalk/webhook/send       
    #警报被解决之后是否通知 消息模板/usr/local/prometheus-webhook-dingtalk/config.yml
    send_resolved: true
 
################结束###############
检测配置是否正确 并 重启
./amtool check-config alertmanager.yml
systemctl restart alertmanager
```
在grafana的alert-admin中添加alertmanager地址\
在grafana的alert-Concat point中添加Alertmanager预警\
在grafana的alert-Policies中使用Alertmanager预警


### 配置altermanager消息模板

#### 邮件消息
``` bash
mkdir template && vi template/email.tmp
################开始###############
{{ define "email.html" }}
    {{ range .Alerts }}
<pre>
    ========start==========
    告警程序: prometheus_alert 
    告警级别: {{ .Labels.severity }} 
    告警类型: {{ .Labels.alertname }} 
    故障主机: {{ .Labels.instance }} 
    告警主题: {{ .Annotations.summary }}
    告警详情: {{ .Annotations.description }}
    触发时间: {{ (.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}
    ========end==========
</pre>
    {{ end }}
{{ end }}
################结束###############
```
#### 钉钉预警信息

## 安装钉钉预警插件

### 下载对应安装包并解压
``` bash
cd /home/package
 
wget https://github.com/timonwong/prometheus-webhook-dingtalk/releases/download/v1.4.0/prometheus-webhook-dingtalk-1.4.0.linux-amd64.tar.gz
 
tar -zxvf prometheus-webhook-dingtalk-1.4.0.linux-amd64.tar.gz -C /usr/local/
 
ln -s /usr/local/prometheus-webhook-dingtalk-1.4.0.linux-amd64/ /usr/local/prometheus-webhook-dingtalk
```
### 配置-钉钉预警-开机自启
``` bash
vi /usr/lib/systemd/system/prometheus-webhook-dingtalk.service

[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target
 
[Service]
ExecStart=/usr/local/prometheus-webhook-dingtalk/prometheus-webhook-dingtalk  --config.file=/usr/local/prometheus-webhook-dingtalk/config.yml
[Install]
WantedBy=default.target
 
启动 && 开机自启动
systemctl start prometheus-webhook-dingtalk
systemctl enable prometheus-webhook-dingtalk
```
### 配置-钉钉预警-报警属性
修改相关配置
``` bash
cd /usr/local/prometheus-webhook-dingtalk && vi config.yml

##################开始###################
# Request timeout
 timeout: 5s
 
## Customizable templates path
# templates:
#   - contrib/templates/legacy/template.tmpl
#模板文件
 templates:
    - /usr/local/prometheus-webhook-dingtalk/template/*.tmp
## You can also override default template using `default_message`
## The following example to use the 'legacy' template from v0.3.0
# default_message:
#   title: '{{ template "legacy.title" . }}'
#   text: '{{ template "legacy.content" . }}'
## Targets, previously was known as "profiles"
# 钉钉群自己创建机器人
 targets:
   webhook:
     url: https://oapi.dingtalk.com/robot/send?access_token=**************
     message:
       # Use legacy template
       title: '{{ template "ding.link.title" . }}'
       text: '{{ template "ding.link.content" . }}'

###############结束#############
```
### 配置-钉钉预警-消息模板
创建模板文件
``` bash 
mkdir template && vi template/template.tmp
 
{{ define "__subject" }}[Linux 基础监控告警:{{ .Alerts.Firing | len }}] {{ end }}
 
{{ define "__text_list" }}{{ range . }}
 
{{ range .Labels.SortedPairs }}
{{ if eq .Name "instance" }}> 实例: {{ .Value | html }}{{ end }}
{{ end }}
 
{{ range .Labels.SortedPairs }}
{{ if eq .Name "serverity" }}> 告警级别: {{ .Value | html }}{{ end }}
{{ if eq .Name "hostname" }}> 主机名称: {{ .Value | html }}{{ end }}
{{ end }}
 
{{ range .Annotations.SortedPairs }}
{{ if eq .Name "description" }}> 告警详情: {{ .Value | html }}{{ end }}
{{ end }}
触发时间: {{ (.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}
 
{{"============================"}}
{{ end }}{{ end }}
 
{{ define "ding.link.title" }}{{ template "__subject" . }}{{ end }}
{{ define "ding.link.content" }}
{{ if gt (len .Alerts.Firing) 0 }}#### [{{ .Alerts.Firing | len }}]【Linux 报警触发】
{{ template "__text_list" .Alerts.Firing }}{{ end }}
{{ if gt (len .Alerts.Resolved) 0 }}#### [{{ .Alerts.Resolved | len }}]【Linux 报警恢复】
{{ end }}
{{ end }}
```

## 编写告警规则
告警规则[参考](https://www.bbsmax.com/A/1O5EQv7G57/)

### 采集规则文件

``` bash
cd /usr/local/prometheus && mkdir rules && cd rules

修改对应位置
vi prometheus.yml
 
# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
           - 127.0.0.1:9093       # 9093为altermanager预警接口，产生预警则向指定端口发送
 
# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
   - "rules/*.yml"
  # - "first_rules.yml"
  # - "second_rules.yml"
 
```
### 编写告警规则
#### 编写记录规则 参考：k8s-record-rules.yml
```bash
groups:
  - name: k8s-record
    rules:
    - expr: node_uname_info{job=~"k8s"}
      record: k8s:node_uname_info
      labels:
        desc: "节点信息"
        unit: " "
        job: "k8s"
    - expr: up{job=~"k8s"}
      record: k8s:up
      labels:
        desc: "节点是否在线, 在线1,不在线0"
        unit: " "
        job: "k8s"
    - expr: time() - node_boot_time_seconds{job=~"k8s"}
      record: k8s:node_uptime
      labels:
        desc: "节点的运行时间"
        unit: "s"
        job: "k8s"
##############################################################################################
#                              cpu                                                           #
    - expr: (1 - avg by (environment,instance) (irate(node_cpu_seconds_total{job="k8s",mode="idle"}[5m])))  * 100
      record: k8s:cpu:total:percent
      labels:
        desc: "节点的cpu总消耗百分比"
        unit: "%"
        job: "k8s"
 
    - expr: (avg by (environment,instance) (irate(node_cpu_seconds_total{job="k8s",mode="idle"}[5m])))  * 100
      record: k8s:cpu:idle:percent
      labels:
        desc: "节点的cpu idle百分比"
        unit: "%"
        job: "k8s"
 
    - expr: (avg by (environment,instance) (irate(node_cpu_seconds_total{job="k8s",mode="iowait"}[5m])))  * 100
      record: k8s:cpu:iowait:percent
      labels:
        desc: "节点的cpu iowait百分比"
        unit: "%"
        job: "k8s"
 
    - expr: (avg by (environment,instance) (irate(node_cpu_seconds_total{job="k8s",mode="system"}[5m])))  * 100
      record: k8s:cpu:system:percent
      labels:
        desc: "节点的cpu system百分比"
        unit: "%"
        job: "k8s"
 
    - expr: (avg by (environment,instance) (irate(node_cpu_seconds_total{job="k8s",mode="user"}[5m])))  * 100
      record: k8s:cpu:user:percent
      labels:
        desc: "节点的cpu user百分比"
        unit: "%"
        job: "k8s"
 
    - expr: (avg by (environment,instance) (irate(node_cpu_seconds_total{job="k8s",mode=~"softirq|nice|irq|steal"}[5m])))  * 100
      record: k8s:cpu:other:percent
      labels:
        desc: "节点的cpu 其他的百分比"
        unit: "%"
        job: "k8s"
##############################################################################################
 
##############################################################################################
#                                    memory                                                  #
    - expr: node_memory_MemTotal_bytes{job="k8s"}
      record: k8s:memory:total
      labels:
        desc: "节点的内存总量"
        unit: byte
        job: "k8s"
 
    - expr: node_memory_MemFree_bytes{job="k8s"}
      record: k8s:memory:free
      labels:
        desc: "节点的剩余内存量"
        unit: byte
        job: "k8s"
 
    - expr: node_memory_MemTotal_bytes{job="k8s"} - node_memory_MemFree_bytes{job="k8s"}
      record: k8s:memory:used
      labels:
        desc: "节点的已使用内存量"
        unit: byte
        job: "k8s"
 
    - expr: node_memory_MemTotal_bytes{job="k8s"} - node_memory_MemAvailable_bytes{job="k8s"}
      record: k8s:memory:actualused
      labels:
        desc: "节点用户实际使用的内存量"
        unit: byte
        job: "k8s"
 
    - expr: ((node_memory_MemTotal_bytes{job="k8s"} - node_memory_MemFree_bytes{job="k8s"} - node_memory_Buffers_bytes{job="k8s"} - node_memory_Cached_bytes{job="k8s"}) / (node_memory_MemTotal_bytes{job="k8s"} )) * 100 
      record: k8s:memory:used:percent
      labels:
        desc: "节点的内存使用百分比"
        unit: "%"
        job: "k8s"
 
    - expr: ((node_memory_MemAvailable_bytes{job="k8s"} / (node_memory_MemTotal_bytes{job="k8s"})))* 100
      record: k8s:memory:free:percent
      labels:
        desc: "节点的内存剩余百分比"
        unit: "%"
        job: "k8s"
##############################################################################################
#                                   load                                                     #
    - expr: sum by (instance) (node_load1{job="k8s"})
      record: k8s:load:load1
      labels:
        desc: "系统1分钟负载"
        unit: " "
        job: "k8s"
 
    - expr: sum by (instance) (node_load5{job="k8s"})
      record: k8s:load:load5
      labels:
        desc: "系统5分钟负载"
        unit: " "
        job: "k8s"
 
    - expr: sum by (instance) (node_load15{job="k8s"})
      record: k8s:load:load15
      labels:
        desc: "系统15分钟负载"
        unit: " "
        job: "k8s"
 
##############################################################################################
#                                 disk                                                       #
    - expr: node_filesystem_size_bytes{job="k8s" ,fstype=~"ext4|xfs"}
      record: k8s:disk:usage:total
      labels:
        desc: "节点的磁盘总量"
        unit: byte
        job: "k8s"
 
    - expr: node_filesystem_avail_bytes{job="k8s",fstype=~"ext4|xfs"}
      record: k8s:disk:usage:free
      labels:
        desc: "节点的磁盘剩余空间"
        unit: byte
        job: "k8s"
 
    - expr: node_filesystem_size_bytes{job="k8s",fstype=~"ext4|xfs"} - node_filesystem_avail_bytes{job="k8s",fstype=~"ext4|xfs"}
      record: k8s:disk:usage:used
      labels:
        desc: "节点的磁盘使用的空间"
        unit: byte
        job: "k8s"
 
    - expr:  (1 - node_filesystem_avail_bytes{job="k8s",fstype=~"ext4|xfs"} / node_filesystem_size_bytes{job="k8s",fstype=~"ext4|xfs"}) * 100
      record: k8s:disk:used:percent
      labels:
        desc: "节点的磁盘的使用百分比"
        unit: "%"
        job: "k8s"
 
    - expr: irate(node_disk_reads_completed_total{job="k8s"}[1m])
      record: k8s:disk:read:count:rate
      labels:
        desc: "节点的磁盘读取速率"
        unit: "次/秒"
        job: "k8s"
 
    - expr: irate(node_disk_writes_completed_total{job="k8s"}[1m])
      record: k8s:disk:write:count:rate
      labels:
        desc: "节点的磁盘写入速率"
        unit: "次/秒"
        job: "k8s"
 
    - expr: (irate(node_disk_written_bytes_total{job="k8s"}[1m]))/1024/1024
      record: k8s:disk:read:mb:rate
      labels:
        desc: "节点的设备读取MB速率"
        unit: "MB/s"
        job: "k8s"
 
    - expr: (irate(node_disk_read_bytes_total{job="k8s"}[1m]))/1024/1024
      record: k8s:disk:write:mb:rate
      labels:
        desc: "节点的设备写入MB速率"
        unit: "MB/s"
        job: "k8s"
 
##############################################################################################
#                                filesystem                                                  #
    - expr:   (1 -node_filesystem_files_free{job="k8s",fstype=~"ext4|xfs"} / node_filesystem_files{job="k8s",fstype=~"ext4|xfs"}) * 100
      record: k8s:filesystem:used:percent
      labels:
        desc: "节点的inode的剩余可用的百分比"
        unit: "%"
        job: "k8s"
#############################################################################################
#                                filefd                                                     #
    - expr: node_filefd_allocated{job="k8s"}
      record: k8s:filefd_allocated:count
      labels:
        desc: "节点的文件描述符打开个数"
        unit: "%"
        job: "k8s"
 
    - expr: node_filefd_allocated{job="k8s"}/node_filefd_maximum{job="k8s"} * 100
      record: k8s:filefd_allocated:percent
      labels:
        desc: "节点的文件描述符打开百分比"
        unit: "%"
        job: "k8s"
 
#############################################################################################
#                                network                                                    #
    - expr: avg by (environment,instance,device) (irate(node_network_receive_bytes_total{device=~"eth0|eth1|ens33|ens37"}[1m]))
      record: k8s:network:netin:bit:rate
      labels:
        desc: "节点网卡eth0每秒接收的比特数"
        unit: "bit/s"
        job: "k8s"
 
    - expr: avg by (environment,instance,device) (irate(node_network_transmit_bytes_total{device=~"eth0|eth1|ens33|ens37"}[1m]))
      record: k8s:network:netout:bit:rate
      labels:
        desc: "节点网卡eth0每秒发送的比特数"
        unit: "bit/s"
        job: "k8s"
 
    - expr: avg by (environment,instance,device) (irate(node_network_receive_packets_total{device=~"eth0|eth1|ens33|ens37"}[1m]))
      record: k8s:network:netin:packet:rate
      labels:
        desc: "节点网卡每秒接收的数据包个数"
        unit: "个/秒"
        job: "k8s"
 
    - expr: avg by (environment,instance,device) (irate(node_network_transmit_packets_total{device=~"eth0|eth1|ens33|ens37"}[1m]))
      record: k8s:network:netout:packet:rate
      labels:
        desc: "节点网卡发送的数据包个数"
        unit: "个/秒"
        job: "k8s"
 
    - expr: avg by (environment,instance,device) (irate(node_network_receive_errs_total{device=~"eth0|eth1|ens33|ens37"}[1m]))
      record: k8s:network:netin:error:rate
      labels:
        desc: "节点设备驱动器检测到的接收错误包的数量"
        unit: "个/秒"
        job: "k8s"
 
    - expr: avg by (environment,instance,device) (irate(node_network_transmit_errs_total{device=~"eth0|eth1|ens33|ens37"}[1m]))
      record: k8s:network:netout:error:rate
      labels:
        desc: "节点设备驱动器检测到的发送错误包的数量"
        unit: "个/秒"
        job: "k8s"
 
    - expr: node_tcp_connection_states{job="k8s", state="established"}
      record: k8s:network:tcp:established:count
      labels:
        desc: "节点当前established的个数"
        unit: "个"
        job: "k8s"
 
    - expr: node_tcp_connection_states{job="k8s", state="time_wait"}
      record: k8s:network:tcp:timewait:count
      labels:
        desc: "节点timewait的连接数"
        unit: "个"
        job: "k8s"
 
    - expr: sum by (environment,instance) (node_tcp_connection_states{job="k8s"})
      record: k8s:network:tcp:total:count
      labels:
        desc: "节点tcp连接总数"
        unit: "个"
        job: "k8s"
 
#############################################################################################
#                                process                                                    #
    - expr: node_processes_state{state="Z"}
      record: k8s:process:zoom:total:count
      labels:
        desc: "节点当前状态为zoom的个数"
        unit: "个"
        job: "k8s"
#############################################################################################
#                                other                                                    #
    - expr: abs(node_timex_offset_seconds{job="k8s"})
      record: k8s:time:offset
      labels:
        desc: "节点的时间偏差"
        unit: "s"
        job: "k8s"
 
#############################################################################################
 
    - expr: count by (instance) ( count by (instance,cpu) (node_cpu_seconds_total{ mode='system'}) )
      record: k8s:cpu:count
#
```
#### 编写告警规则 参考：k8s-alert-rules.yml
``` bash
groups:
  - name: k8s-alert
    rules:
    - alert: k8s-down
      expr: k8s:up == 0
      for: 1m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} 宕机了"
        description: "instance: {{ $labels.instance }} \n- job: {{ $labels.job }} 宕机了， 时间已经超过1分钟了。"
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"
        grafana: "http://monitor.yurun.com/d/9CWBz0bik/fu-wu-qi-jian-kong-mian-ban?orgId=1&var-job=k8s"
        id: "{{ $labels.instanceid }}"
 
    - alert: k8s-cpu-high
      expr:  k8s:cpu:total:percent > 80
      for: 3m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} cpu 使用率高于 {{ $value }}"
        description: "{{ $labels.instance }} of {{$labels.job}} cpu使用率超过80%,当前使用率[{{ $value }}]."
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"
        grafana: "http://monitor.yurun.com/d/9CWBz0bik/fu-wu-qi-jian-kong-mian-ban?orgId=1&var-job=k8s"
        id: "{{ $labels.instanceid }}"
 
    - alert: k8s-cpu-iowait-high
      expr:  k8s:cpu:iowait:percent >= 12
      for: 3m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} cpu iowait 使用率高于 {{ $value }}"
        description: "{{ $labels.instance }} of {{$labels.job}} cpu-iowait使用率超过12%,当前使用率[{{ $value }}]%."
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"
        grafana: "http://monitor.yurun.com/d/9CWBz0bik/fu-wu-qi-jian-kong-mian-ban?orgId=1&var-job=k8s"
        id: "{{ $labels.instanceid }}"
 
    - alert: k8s-load-load1-high
      expr:  (k8s:load:load1) > (k8s:cpu:count) * 1.2
      for: 3m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} 系统1分钟负载高于((k8s:cpu:count) * 1.2)"
        description: "{{ $labels.instance }} of {{$labels.job}} 系统1分钟负载超过((k8s:cpu:count) * 1.2)%,当前使用率[{{ $value }}]%."
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"
        grafana: "http://monitor.yurun.com/d/9CWBz0bik/fu-wu-qi-jian-kong-mian-ban?orgId=1&var-job=k8s"
        id: "{{ $labels.instanceid }}"
 
    - alert: k8s-memory-high
      expr:  k8s:memory:used:percent > 85
      for: 3m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} 内存使用率高于 {{ $value }}"
        description: "{{ $labels.instance }} of {{$labels.job}} 内存使用率超过85%,当前使用率[{{ $value }}]%."
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"
        grafana: "http://monitor.yurun.com/d/9CWBz0bik/fu-wu-qi-jian-kong-mian-ban?orgId=1&var-job=k8s"
        id: "{{ $labels.instanceid }}"
 
    - alert: k8s-disk-high
      expr:  k8s:disk:used:percent > 88
      for: 10m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} 磁盘使用率高于 {{ $value }}"
        description: "{{ $labels.instance }} of {{$labels.job}} 磁盘使用率超过88%,当前使用率[{{ $value }}]%."
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"
        grafana: "http://monitor.yurun.com/d/9CWBz0bik/fu-wu-qi-jian-kong-mian-ban?orgId=1&var-job=k8s"
        id: "{{ $labels.instanceid }}"
 
    - alert: k8s-disk-read:count-high
      expr:  k8s:disk:read:count:rate > 3000
      for: 2m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} 磁盘读取速率高于 {{ $value }}次/秒"
        description: "{{ $labels.instance }} of {{$labels.job}} 磁盘读取速率超过3000次/秒,当前使用率[{{ $value }}]次/秒."
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"
        grafana: "http://monitor.yurun.com/d/9CWBz0bik/fu-wu-qi-jian-kong-mian-ban?orgId=1&var-job=k8s"
        id: "{{ $labels.instanceid }}"
 
    - alert: k8s-disk-write-count-high
      expr:  k8s:disk:write:count:rate > 3000
      for: 2m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} iops write 磁盘写入速率高于 {{ $value }}"
        description: "{{ $labels.instance }} of {{$labels.job}} 磁盘写入速率超过3000次/秒,当前使用率[{{ $value }}]次/秒."
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"
        grafana: "http://monitor.yurun.com/d/9CWBz0bik/fu-wu-qi-jian-kong-mian-ban?orgId=1&var-job=k8s"
        id: "{{ $labels.instanceid }}"
 
    - alert: k8s-disk-read-mb-high
      expr:  k8s:disk:read:mb:rate > 60
      for: 2m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} 设备读取MB速率高于 {{ $value }}"
        description: "{{ $labels.instance }} of {{$labels.job}} 设备读取MB速率超过60MB/s,当前使用率[{{ $value }}]MB/s."
        instance: "{{ $labels.instance }}"
        value: "{{ $value }}"
        grafana: "http://monitor.yurun.com/d/9CWBz0bik/fu-wu-qi-jian-kong-mian-ban?orgId=1&var-job=k8s"
        id: "{{ $labels.instanceid }}"
 
    - alert: k8s-disk-write-mb-high
      expr:  k8s:disk:write:mb:rate > 60
      for: 2m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} 设备写入MB速率高于 {{ $value }}"
        description: "{{ $labels.instance }} of {{$labels.job}} 设备写入MB速率超过60MB/s,当前使用率[{{ $value }}]MB/s."
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"
        grafana: "http://monitor.yurun.com/d/9CWBz0bik/fu-wu-qi-jian-kong-mian-ban?orgId=1&var-job=k8s"
        id: "{{ $labels.instanceid }}"
 
    - alert: k8s-filefd-allocated-percent-high
      expr:  k8s:filefd_allocated:percent > 80
      for: 10m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} 打开文件描述符高于 {{ $value }}"
        description: "{{ $labels.instance }} of {{$labels.job}} 打开文件描述符超过80%,当前使用率[{{ $value }}]%."
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"
        grafana: "http://monitor.yurun.com/d/9CWBz0bik/fu-wu-qi-jian-kong-mian-ban?orgId=1&var-job=k8s"
        id: "{{ $labels.instanceid }}"
 
    - alert: k8s-network-netin-error-rate-high
      expr:  k8s:network:netin:error:rate > 4
      for: 1m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} 接收错误包的数量高于 {{ $value }}"
        description: "{{ $labels.instance }} of {{$labels.job}} 接收错误包的数量超过4个/秒,当前错误包的数量[{{ $value }}]个/秒."
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"
        grafana: "http://monitor.yurun.com/d/9CWBz0bik/fu-wu-qi-jian-kong-mian-ban?orgId=1&var-job=k8s"
        id: "{{ $labels.instanceid }}"

    - alert: k8s-network-netin-packet-rate-high
      expr:  k8s:network:netin:packet:rate > 35000
      for: 1m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} 包进入速率高于 {{ $value }}"
        description: "{{ $labels.instance }} of {{$labels.job}} 包进入速率超过35000个/秒,当前错误包的数量[{{ $value }}]个/秒."
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"
        grafana: "http://monitor.yurun.com/d/9CWBz0bik/fu-wu-qi-jian-kong-mian-ban?orgId=1&var-job=k8s"
        id: "{{ $labels.instanceid }}"
 
    - alert: k8s-network-netout-packet-rate-high
      expr:  k8s:network:netout:packet:rate > 35000
      for: 1m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} 包流出速率 高于 {{ $value }}"
        description: "{{ $labels.instance }} of {{$labels.job}} 包流出速率超过35000个/秒,当前错误包的数量[{{ $value }}]个/秒."
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"
        grafana: "http://monitor.yurun.com/d/9CWBz0bik/fu-wu-qi-jian-kong-mian-ban?orgId=1&var-job=k8s"
        id: "{{ $labels.instanceid }}"

```
### 重启服务prometheus
``` bash
重启服务
systemctl restart prometheus
```