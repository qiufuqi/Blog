---
title: Prometheus监控之blackbox_exporter
date: 2023-12-11
tags:
  - Prometheus
  - 黑盒监控
categories: 
- 运维
- 监控
- 黑盒监控
keywords: 'Prometheus,黑盒监控'
description: Prometheus监控,黑盒监控
cover: https://qiufuqi.github.io/img/hexo/20231213084657.png
abbrlink: prometheus_blackbox
url: prometheus_blackbox
comments: false
top: 999
---

**白盒指标:** 通常Prometheus通过Exporter抓取的指标都是白盒指标监控；
**黑盒指标:** 一种直接能够直接模拟用户访问验证服务的外部可见性的指标获取；
与白盒测试、黑盒测试的对比类似，黑盒测试只关注结果，不关注其内在实现。而黑盒指标监控，也只关心返回的结果是否是我预期的，不关注其内在如何判断是否达到预期了。

黑盒监控和白盒监控：
- 黑盒监控，关注的是实时状态，一般都是正在发生的事件，比如网站访问不了、磁盘无法写入数据等。即黑盒监控的重点是能对正在发生的故障进行告警。常见的黑盒监控包括HTTP探针、TCP探针等用于检测站点或者服务的可访问性，以及访问效率等。
- 白盒监控，关注的是原因，也就是系统内部的一些运行指标数据，例如nginx响应时长、存储I/O负载等

blackbox-exporter是Prometheus官方提供的一个黑盒监控解决方案，可以通过HTTP、HTTPS、DNS、ICMP、TCP和gRPC方式对目标实例进行检测。可用于以下使用场景：
- HTTP/HTTPS：URL/API可用性检测
- ICMP：主机存活检测
- TCP：端口存活检测
- DNS：域名解析
  
监控系统要能够有效的支持百盒监控和黑盒监控，通过白盒能够了解系统内部的实际运行状态，以及对监控指标的观察能够预判出可能出现的潜在问题，从而对潜在的不确定因素进行提前处理避免问题发生；而通过黑盒监控，可以在系统或服务发生故障时快速通知相关人员进行处理。

**部署黑盒监控**
blackbox-exporter[监控部署步骤参考](https://blog.csdn.net/weixin_43266367/article/details/129110541)
原理：安装blackbox-exporter的服务器通过配置发出各种检查信息，比如检测网站，端口等，收集到信息后统一交由prometheus展示，所以一般部署在和prometheus同一台服务器上，也可以部署单独一台服务器（所有检查请求由该服务器发出）

## 安装blackbox-exporter
[文件下载地址1](https://github.com/prometheus/blackbox_exporter)
[文件下载地址2](https://prometheus.io/download/#mysqld_exporter)
### 脚本安装blackbox
blackbox_exporter.sh一键监控安装脚本，提前下载好安装文件或者[在线下载](https://github.com/prometheus/blackbox_exporter/releases)
```bash
#!/bin/bash
# -*- coding: utf-8 -*-
# Date: 2023/12/13
echo "download blackbox_exporter"
sleep 2
wget -N -P /root/ https://github.com/prometheus/blackbox_exporter/releases/download/v0.24.0/blackbox_exporter-0.24.0.linux-amd64.tar.gz
 

echo "tar blackbox_exporter"
sleep 2
tar -zxvf /root/blackbox_exporter-0.24.0.linux-amd64.tar.gz -C /opt/ && mv /opt/blackbox_exporter-0.24.0.linux-amd64 /usr/local/blackbox_exporter

echo "delete blackbox_exporter***tar.gz"
rm -rf /root/blackbox_exporter-0.24.0.linux-amd64.tar.gz


echo "firewall blackbox_exporter port 9115"
sleep 2
firewall-cmd --zone=public --add-port=9115/tcp --permanent && firewall-cmd --reload 
 
echo "add blackbox_exporter.service"
sleep 2
cat << EOF > /usr/lib/systemd/system/blackbox_exporter.service
[Unit]
Description=Prometheus Blackbox Exporter
After=network.target

[Service]
Restart=on-failure
ExecStart=/usr/local/blackbox_exporter/blackbox_exporter --config.file=/usr/local/blackbox_exporter/blackbox.yml --web.listen-address=:9115 --timeout-offset=2
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
 
echo "start blackbox_exporter.service"
sleep 2
systemctl daemon-reload && systemctl start blackbox_exporter && systemctl enable --now blackbox_exporter
```


### 文件安装blackbox
安装blackbox-exporter，采集机器运行数据信息，默认端口9115 （可更改为指定端口），默认0.5秒获取一次数据
blackbox-exporter的配置文件使用默认的即可（/usr/local/blackbox_exporter/blackbox.yml）
```bash
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.24.0/blackbox_exporter-0.24.0.linux-amd64.tar.gz

#解压至指定文件夹
mkdir /usr/local/blackbox_exporter
tar -zxvf blackbox_exporter-0.24.0.linux-amd64.tar.gz -C /usr/local/blackbox_exporter

cd /usr/local/blackbox_exporter
cp blackbox_exporter-0.24.0.linux-amd64/* . && rm -rf blackbox_exporter-0.24.0.linux-amd64

#可查看帮助信息
[root@localhost local]# cd /usr/local/blackbox_exporter
[root@localhost local]# /usr/local/blackbox_exporter/blackbox_exporter -h	

# 下述命令中，--nginx.scrape-uri参数指定了收集指标信息的URI地址，此处的地址是Nginx的状态页。
[root@localhost local]# ./blackbox_exporter --web.listen-address=:9115 

# 配置node_exporter开机自启
vi /usr/lib/systemd/system/blackbox_exporter.service
# 写入以下信息：
[Unit]
Description=Prometheus Blackbox Exporter
After=network.target

[Service]
Restart=on-failure
ExecStart=/usr/local/blackbox_exporter/blackbox_exporter --config.file=/usr/local/blackbox_exporter/blackbox.yml --web.listen-address=:9115 --timeout-offset=2
Restart=on-failure

[Install]
WantedBy=multi-user.target

#重启服务
systemctl daemon-reload
systemctl start blackbox_exporter
systemctl status blackbox_exporter
systemctl enable blackbox_exporter
```
blackbox-exporter的配置文件使用默认的即可（/usr/local/blackbox_exporter/blackbox.yml），文件里定义了进行目标检测时要使用的模块和模块参数。至于要检测哪些目标是定义在Prometheus 的Job配置中。
默认是以下模块和参数，可以添加自己需要的
- http_2xx:http检测，GET方式
- http_post_2xx:http检测，POST方式
- tcp_connect:端口检测
- pop3s_banner:
- grpc:
- grpc_plain:
- ssh_banner:
- irc_banner:
- icmp:实现ICMP监控
- icmp_ttl5:

```bash
modules:
  http_2xx:
    prober: http
    http:
      preferred_ip_protocol: "ip4"
  http_post_2xx:
    prober: http
    http:
      method: POST
  tcp_connect:
    prober: tcp
  pop3s_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^+OK"
      tls: true
      tls_config:
        insecure_skip_verify: false
  grpc:
    prober: grpc
    grpc:
      tls: true
      preferred_ip_protocol: "ip4"
  grpc_plain:
    prober: grpc
    grpc:
      tls: false
      service: "service1"
  ssh_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^SSH-2.0-"
      - send: "SSH-2.0-blackbox-ssh-check"
  irc_banner:
    prober: tcp
    tcp:
      query_response:
      - send: "NICK prober"
      - send: "USER prober prober prober :prober"
      - expect: "PING :([^ ]+)"
        send: "PONG ${1}"
      - expect: "^:[^ ]+ 001"
  icmp:
    prober: icmp
  icmp_ttl5:
    prober: icmp
    timeout: 5s
    icmp:
      ttl: 5
```

## 运行blackbox-exporter
浏览器运行访问 http://10.11.8.108:9115
![](https://qiufuqi.github.io/img/hexo/20231211172546.png)

## 加入prometheus监控
登录prometheus所在服务器，在文件的最下面添加job 配置，并重启Prometheus
```bash
# 配置参考：
- job_name: http-status
    metrics_path: /probe	#指定指标接口
    scrape_interval: 5s
    params:	#指定查询参数，在Prometheus向target发送Get请求获取指标数据时，会传递到url上
      module: [http_2xx]

    static_configs:     # 静态部署
    - targets:    # 可以每行写，也可以写成 ['http://www.baidu.com','http://www.baidu.com']
      - http://www.baidu.com
      - http://www.baidu.com
      labels:	#自定义标签，附加在target上
        group: web

    file_sd_configs:    # 基于文件服务发现
      - files:
        - "./device/http_device.yml"   # - "./device/http_device.json"  yml和json格式选择一个即可
        refresh_interval: 5s # 每隔5秒检查刷新一次服务发现配置

    relabel_configs:
    - source_labels: [__address__]      #将标签__address__的值赋值给__param_target标签，以__param开头的标签也会作为查询参数传递Prometheus的Get请求，作用和上面的params配置类似
      target_label: __param_target
    - source_labels: [__param_target]   #将标签__param_target的值赋值给instance标签
      target_label: instance
    - target_label: __address__         #将标签__address__的值修改给balckbox-expoter的地址
      replacement: 10.11.8.108:9115
   #以 http://www.baidu.com为例，最后其对应target的地址就是http://10.11.8.108:9115/probe?module=http_2xx&target=http://www.baidu.com
```
### http_2xx监控
此job配置了对http://www.baidu.com，验证是否可以访问，这里使用一个blackbox-exporter检测多个目标网站。
对应target的地址就是http://10.11.8.108:9115/probe?module=http_2xx&target=http://www.baidu.com

#### 静态部署
```bash
[root@prometheus ~]# vi /usr/local/prometheus/prometheus.yml
······
  - job_name: 'Http_2xx_get'
    scrape_interval: 5s
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
        - targets: ['https://pig.yurun.com','https://mpig.yurun.com']
          labels:
            instance: web_status
            group: 'web'
    relabel_configs:
        - source_labels: [__address__]
          target_label: __param_target
        - source_labels: [__param_target]
          target_label: instance
        - target_label: __address__
          replacement: 10.11.8.108:9115 # blackbox-exporter 地址和端口
······
# curl接口测试 出现以下信息表示数据正常
[root@prometheus ~]# curl http://10.11.8.108:9115/probe?target=www.baidu.com&module=http_2xx&debug=true
·········
probe_ip_protocol 4
# HELP probe_success Displays whether or not the probe was a success
# TYPE probe_success gauge
probe_success 1

# 重启Prometheus服务，并访问prometheus所在的target状态，如果为UP说明job设置成功：
[root@prometheus ~]# cd /usr/local/prometheus
[root@prometheus prometheus]# ./promtool check config prometheus.yml
[root@prometheus ~]# systemctl restart prometheus
# 保持热加载
[root@prometheus ~]# curl  -X POST localhost:9090/-/reload
```
#### 文件部署
当有新的节点时，只需要修改对应的yml或者json文件即可。
```bash
vi /usr/local/prometheus/prometheus.yml
······
  # 文件部署
  - job_name: 'Http_2xx_get'
    scrape_interval: 5s
    metrics_path: /probe
    file_sd_configs: # 基于文件服务发现
      - files:
        - "./device/http_device.yml"   # - "./device/http_device.json"  yml和json格式选择一个即可
        refresh_interval: 5s # 每隔5秒检查刷新一次服务发现配置
    params:
      module: [http_2xx]
    relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - source_labels: [module]
      target_label: __param_module
    - target_label: __address__
      replacement: 10.11.8.108:9115 # blackbox-exporter 地址和端口


 vi /usr/local/prometheus/device/http_device.yml
 # 或者单独列
- labels:
    group: web
    module: http_2xx     # 不同label可使用不同model，比如，prometheus.yml中传递对应值
  targets:
    - 'https://pig.yurun.com'
    - 'https://mpig.yurun.com'

# 写一起
- labels:
    group: web
    module: http_2xx     # 不同label可使用不同model，比如，prometheus.yml中传递对应值
  targets: ['https://pig.yurun.com','https://mpig.yurun.com']


# json模式  file_sd_configs中同步修改
 vi /usr/local/prometheus/device/http_device.json
[{
	"labels": {
		"group": "web",
		"module": "http_2xx"
	},
	"targets": [
		"https://pig.yurun.com",
		"https://mpig.yurun.com"
	]
}]

······
# 重启Prometheus服务，并访问prometheus所在的target状态，如果为UP说明job设置成功：
[root@prometheus ~]# cd /usr/local/prometheus
[root@prometheus prometheus]# ./promtool check config prometheus.yml
# 保持热加载
[root@prometheus ~]# curl  -X POST localhost:9090/-/reload
```

访问blackbox-exporter所在的服务器http://10.11.8.108:9115/，可以看到以下信息
![](https://qiufuqi.github.io/img/hexo/20231212103830.png)
![](https://qiufuqi.github.io/img/hexo/20231212103746.png)

### icmp监控
此job配置了对112.4.152.2和36.152.156.108，验证是否ping通，这里使用一个blackbox-exporter检测多个IP。
对应target的地址就是http://10.11.8.108:9115/probe?module=icmp&target=36.152.156.108
#### 静态部署
```bash
[root@prometheus ~]# vi /usr/local/prometheus/prometheus.yml
······
  - job_name: 'Icmp_ping'
    scrape_interval: 5s
    metrics_path: /probe
    params:
      module: [icmp]
    static_configs:
        - targets: ['112.4.152.2','36.152.156.108']
          labels:
            instance: icmp_status
            group: 'icmp'
    relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: 10.11.8.108:9115
······
# 重启Prometheus服务，并访问prometheus所在的target状态，如果为UP说明job设置成功：
[root@prometheus ~]# cd /usr/local/prometheus
[root@prometheus prometheus]# ./promtool check config prometheus.yml
[root@prometheus ~]# systemctl restart prometheus
# 保持热加载
[root@prometheus ~]# curl  -X POST localhost:9090/-/reload
```
#### 文件部署
当有新的节点时，只需要修改对应的yml或者json文件即可。
```bash
vi /usr/local/prometheus/prometheus.yml
······
  # 文件部署
  - job_name: 'Icmp_ping'
    scrape_interval: 5s
    metrics_path: /probe
    file_sd_configs: # 基于文件服务发现
      - files:
        - "./device/icmp_device.yml"   # - "./device/icmp_device.json"  yml和json格式选择一个即可
        refresh_interval: 5s # 每隔5秒检查刷新一次服务发现配置
    params:
      module:
        - icmp
    relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - source_labels: [__param_module]
      target_label: module
    - target_label: __address__
      replacement: 10.11.8.108:9115

 vi /usr/local/prometheus/device/icmp_device.yml
 # 或者单独列
- labels:
    instance: icmp_status
    group: icmp
    module: icmp
  targets:
    - '112.4.152.2'
    - '36.152.156.108'

# 写一起
- labels:
    instance: icmp_status
    group: icmp
    module: icmp
  targets: ['112.4.152.2','36.152.156.108']


# json模式  file_sd_configs中同步修改
 vi /usr/local/prometheus/device/icmp_device.json
[{
	"labels": {
		"instance": "icmp_status",
		"group": "icmp",
    "module": "icmp"
	},
	"targets": [
		"112.4.152.2",
		"136.152.156.108"
	]
}]

······
# 重启Prometheus服务，并访问prometheus所在的target状态，如果为UP说明job设置成功：
[root@prometheus ~]# cd /usr/local/prometheus
[root@prometheus prometheus]# ./promtool check config prometheus.yml
# 保持热加载
[root@prometheus ~]# curl  -X POST localhost:9090/-/reload
```
访问blackbox-exporter所在的服务器http://10.11.8.108:9115/，可以看到以下信息
![](https://qiufuqi.github.io/img/hexo/20231212110928.png)
![](https://qiufuqi.github.io/img/hexo/20231212110948.png)

### tcp_connect监控
此job配置了对10.11.7.216:7190和10.11.7.216:52089，端口监控，这里使用一个blackbox-exporter检测多个IP。
对应target的地址就是http://10.11.8.108:9115/probe?module=tcp_connect&target=10.11.7.216:7190
#### 静态部署
```bash
[root@prometheus ~]# vi /usr/local/prometheus/prometheus.yml
······
  - job_name: 'Tcp_connect'
    scrape_interval: 5s
    metrics_path: /probe
    params:
      module: [tcp_connect]
    static_configs:
        - targets: ['10.11.7.216:7190','10.11.7.216:52089']
          labels:
            instance: tcp_status
            group: 'tcp'
    relabel_configs:
        - source_labels: [__address__]
          target_label: __param_target
        - source_labels: [__param_target]
          target_label: instance
        - target_label: __address__
          replacement: 10.11.8.108:9115
······

# 重启Prometheus服务，并访问prometheus所在的target状态，如果为UP说明job设置成功：
[root@prometheus ~]# cd /usr/local/prometheus
[root@prometheus prometheus]# ./promtool check config prometheus.yml
[root@prometheus ~]# systemctl restart prometheus
# 保持热加载
[root@prometheus ~]# curl  -X POST localhost:9090/-/reload
```
#### 文件部署
当有新的节点时，只需要修改对应的yml或者json文件即可。
```bash
vi /usr/local/prometheus/prometheus.yml
······
  # 文件部署
  - job_name: 'Tcp_connect'
    scrape_interval: 5s
    metrics_path: /probe
    file_sd_configs: # 基于文件服务发现
      - files:
        - "./device/tcp_device.yml"   # - "./device/tcp_device.json"  yml和json格式选择一个即可
        refresh_interval: 5s # 每隔5秒检查刷新一次服务发现配置
    params:
      module:
        - tcp_connect
    relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - source_labels: [__param_module]
      target_label: module
    - target_label: __address__
      replacement: 10.11.8.108:9115

 vi /usr/local/prometheus/device/tcp_device.yml
 # 或者单独列
- labels:
    instance: tcp_status
    group: tcp
    module: tcp_connect
  targets:
    - '10.11.7.216:7190'
    - '10.11.7.216:52089'

# 写一起
- labels:
    instance: tcp_status
    group: tcp
    module: tcp_connect
  targets: ['10.11.7.216:7190','10.11.7.216:52089']


# json模式  file_sd_configs中同步修改
 vi /usr/local/prometheus/device/tcp_device.json
[{
	"labels": {
		"instance": "tcp_status",
    "group": "tcp",
		"module": "tcp_connect"
	},
	"targets": [
		"10.11.7.216:7190",
		"10.11.7.216:52089"
	]
}]

······
# 重启Prometheus服务，并访问prometheus所在的target状态，如果为UP说明job设置成功：
[root@prometheus ~]# cd /usr/local/prometheus
[root@prometheus prometheus]# ./promtool check config prometheus.yml
# 保持热加载
[root@prometheus ~]# curl  -X POST localhost:9090/-/reload
```
访问blackbox-exporter所在的服务器http://10.11.8.108:9115/，可以看到以下信息
![](https://qiufuqi.github.io/img/hexo/20231212114617.png)
![](https://qiufuqi.github.io/img/hexo/20231212114652.png)

### grafa页面展示
[面板展示大全](https://grafana.com/grafana/dashboards/)
推荐使用13659
![](https://qiufuqi.github.io/img/hexo/20231212155927.png)

面板id:11175展示网站状态
![](https://qiufuqi.github.io/img/hexo/20231212143024.png)

面板id:12275 展示
![](https://qiufuqi.github.io/img/hexo/20231212135806.png)

面板id:9965 展示
![](https://qiufuqi.github.io/img/hexo/20231212135412.png)

面板id:13230 展示SSL证书状态
![](https://qiufuqi.github.io/img/hexo/20231212135454.png)

## 添加告警规则
添加相关告警规则，[告警规则参考](https://huaweicloud.csdn.net/638db19bdacf622b8df8c612.html#blackbox_exporter_372)
```bash
# 主机端口不通
expr: probe_success{job="Tcp_connect"} == 0
# 主机ping不通
expr: probe_success{job="Icmp_ping"} == 0
# 非200HTTP状态码
expr: probe_http_status_code{job="blackbox_http_2xx"} != 200
# SSL证书还有30天过期
expr: probe_ssl_earliest_cert_expiry{job="Http_2xx_get"} - time() < 86400 * 30
# 接口总耗时大于 3 秒的告警。
expr: sum(probe_http_duration_seconds) by (instance) > 3  
```
编写告警规则 参考：BlackBox-alert-rules.yml
建议原样拷贝，格式很重要，可以通过命令行检测
/usr/local/prometheus/promtool check config prometheus.yml
```bash
groups:
  - name: blackbox-probe-alert
    rules:
    - alert: probeHttpStatus
      expr: probe_success{job="Icmp_ping"}==0
      for: 1m
      labels:
        severity: red
      annotations:
        summary: '节点不可达，请及时查看'
        description: '{{$labels.instance}} 节点不可达,请及时查看'
        resolvetion: "{{$labels.instance}} 节点已恢复."
  - name: blackbox-port-alert
    rules:
    - alert: probeHttpPort
      expr: probe_success{job="Tcp_connect"}==0
      for: 1m
      labels:
        severity: red
      annotations:
        summary: '节点不可达，请及时查看'
        description: '{{$labels.instance}} 节点端口不可达,请及时查看'
        resolvetion: "{{$labels.instance}} 节点端口已恢复."
  - name: blackbox-http-alert
    rules:
    - alert: curlHttpStatus
      expr: probe_http_status_code{job="Http_2xx_get"} >=400 and probe_success{job="Http_2xx_get"}==0
      for: 1m
      labels:
        severity: red
      annotations:
        summary: 'web接口访问异常状态码 > 400'
        description: '{{$labels.instance}} 不可访问,请及时查看,当前状态码为{{$value}}'
        resolvetion: "{{$labels.instance}} 访问已恢复."
  - name: blackbox-http-time-alert
    rules:
    - alert: curlHttpStatus
      expr: sum(probe_http_duration_seconds) by (instance) > 3
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: 'web接口访问总耗时大于 3 秒'
        description: '{{$labels.instance}} 接口访问总耗时大于 3 秒'
        resolvetion: "{{$labels.instance}} 接口访问时间已恢复，低于 3 秒."
  - name: blackbox-ssl_expiry
    rules:
    - alert: Ssl Cert Will Expire in 30 days
      expr: probe_ssl_earliest_cert_expiry - time() < 86400 * 30
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "域名证书即将过期 (instance {{ $labels.instance }})"
        description: "域名证书 30 天后过期 \n  LABELS: {{ $labels }}"
        resolvetion: "域名证书已延期."
```