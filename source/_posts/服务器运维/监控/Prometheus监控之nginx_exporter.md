---
title: Prometheus监控之Nginx_exporter
date: 2023-12-07
tags:
  - Prometheus
  - Nginx
categories: 
- 运维
- 监控
- Nginx
keywords: 'Prometheus,中间件,Nginx'
description: Prometheus监控,Nginx
cover: https://qiufuqi.github.io/img/hexo/20231213151555.png
abbrlink: prometheus_nginx
url: prometheus_nginx
comments: false
top: 998
---
Prometheus监控Nginx有两种方式。一种是通过nginx_exporter监控，需要开启nginx_stub_status，主要是nginx自身的status信息，metrics数据相对较少；另一种是使用nginx-vts-exporter监控，但是需要在编译nginx的时候添加nginx-module-vts模块，监控数据较多，提供了包含server、upstream以及cache的相关监控指标，指标更加丰富。
# nginx_exporter 监控
Nginx[监控部署步骤参考](https://cloud.tencent.com/document/product/1416/56039)
## 添加stub_status
我们Web 部署在 nginx 配置中，需要在对应的配置文件中进行 stub_status 模块的设置，才能被 exporter 识别。
nginx 配置文件添加下面代码：
```bash
[root@localhost conf.d]# vi /etc/nginx/conf.d/test.conf
server {
        listen 80;
        server_name 127.0.0.1;
        location /stub_status {
                stub_status on;
                access_log off;
        }
}
```
## 访问stub_status
修改配置后，重启nginx，访问 http://ip:port/stub_status ,出现下图界面，说明配置生效：
![](https://qiufuqi.github.io/img/hexo/20231205163352.png)
## 安装nginx-exporter
### 脚本安装nginx_exporter
nginx_exporter.sh一键监控安装脚本，提前下载好安装文件或者[在线下载](https://github.com/nginxinc/nginx-prometheus-exporter/releases)
```bash
#!/bin/bash
# -*- coding: utf-8 -*-
# Date: 2023/12/13
echo "download nginx_prometheus_exporter"
sleep 2
wget -N -P /root/ https://github.com/nginxinc/nginx-prometheus-exporter/releases/download/v1.0.0/nginx-prometheus-exporter_1.0.0_linux_amd64.tar.gz
 

echo "tar nginx_prometheus_exporter"
sleep 2
mkdir /usr/local/nginx_prometheus_exporter
tar -zxvf /root/nginx-prometheus-exporter_1.0.0_linux_amd64.tar.gz -C /usr/local/nginx_prometheus_exporter

echo "delete nginx_prometheus_exporter***tar.gz"
rm -rf /root/nginx-prometheus-exporter_1.0.0_linux_amd64.tar.gz

echo "firewall nginx_prometheus_exporter port 9113"
sleep 2
firewall-cmd --zone=public --add-port=9113/tcp --permanent && firewall-cmd --reload 
 
echo "add nginx_prometheus_exporter.service"
sleep 2
cat << EOF > /usr/lib/systemd/system/nginx_prometheus_exporter.service
[Unit]
Description=nginx_prometheus_exporter
After=network.target 
 
[Service]
Restart=on-failure
ExecStart=/usr/local/nginx_prometheus_exporter/nginx-prometheus-exporter --web.listen-address=:9113 --nginx.scrape-uri=http://127.0.0.1:80/stub_status

[Install]
WantedBy=multi-user.target
EOF
 
echo "start nginx_prometheus_exporter.service"
sleep 2
systemctl daemon-reload && systemctl start nginx_prometheus_exporter && systemctl enable --now nginx_prometheus_exporter
```
### 文件安装exporter
[文件下载地址](https://github.com/nginxinc/nginx-prometheus-exporter/releases)
安装node_exporter，采集机器运行数据信息，默认端口9113 （可更改为指定端口）
```bash
wget https://github.com/nginxinc/nginx-prometheus-exporter/releases/download/v1.0.0/nginx-prometheus-exporter_1.0.0_linux_amd64.tar.gz

#解压至指定文件夹
mkdir /usr/local/nginx_prometheus_exporter
tar -zxvf nginx-prometheus-exporter_1.0.0_linux_amd64.tar.gz -C /usr/local/nginx_prometheus_exporter

#可查看命令
[root@localhost local]# cd /usr/local/nginx_prometheus_exporter
[root@localhost local]# ./nginx-prometheus-exporter --help

# 下述命令中，--nginx.scrape-uri参数指定了收集指标信息的URI地址，此处的地址是Nginx的状态页。
[root@localhost local]# ./nginx-prometheus-exporter --web.listen-address=:9113 --nginx.scrape-uri=http://127.0.0.1:80/stub_status

# 配置node_exporter开机自启
vi /usr/lib/systemd/system/nginx_prometheus_exporter.service
# 写入以下信息：
[Unit]
Description=nginx_prometheus_exporter
After=network.target 
 
[Service]
Restart=on-failure
ExecStart=/usr/local/nginx_prometheus_exporter/nginx-prometheus-exporter --web.listen-address=:9113 --nginx.scrape-uri=http://127.0.0.1:80/stub_status

[Install]
WantedBy=multi-user.target

#重启服务
systemctl daemon-reload
systemctl start nginx_prometheus_exporter
systemctl status nginx_prometheus_exporter
systemctl enable nginx_prometheus_exporter
```
### docker安装exporter
命令参考：docker run -p 9113:9113 -d –restart=always –name nginx-exporter nginx/nginx-prometheus-exporter:版本 -nginx.scrape-uri http://:端口/stub_status
stub_status为上文配置的命名，可同步修改，–restart=always开机自启动
```bash
# 指定版本
docker run -p 9113:9113 nginx/nginx-prometheus-exporter:0.8.0 -nginx.scrape-uri=http://10.11.8.108:80/stub_status

# 最新版本
docker pull nginx/nginx-prometheus-exporter
docker run -p 9113:9113 -d  --restart=always --name nginx-exporter nginx/nginx-prometheus-exporter -nginx.scrape-uri http://10.11.8.108:80/stub_status
```
## 访问nginx-exporter
浏览器运行访问 http://10.11.8.108:9113/metrics
![](https://qiufuqi.github.io/img/hexo/20231205165409.png)
## 加入prometheus监控
登录prometheus所在服务器，在文件的最下面添加job 配置，并重启Prometheus
### 静态部署
```bash
[root@prometheus ~]# vi /usr/local/prometheus/prometheus.yml
······
  # 静态部署，每次修改需要重启
  - job_name: 'Nginx'
    scrape_interval: 5s
    static_configs:
      - targets: ['10.11.7.52:9113','10.128.2.102:9113']
······
# 重启Prometheus服务，并访问prometheus所在的target状态，如果为UP说明job设置成功：
[root@prometheus ~]# cd /usr/local/prometheus
[root@prometheus prometheus]# ./promtool check config prometheus.yml
[root@prometheus ~]# systemctl restart prometheus
# 保持热加载
[root@prometheus ~]# curl  -X POST localhost:9090/-/reload
```
### 文件部署
当有新的节点时，只需要修改对应的yml或者json文件即可。
```bash
vi /usr/local/prometheus/prometheus.yml
······
  # 文件部署
  - job_name: 'Nginx'
    scrape_interval: 5s
    file_sd_configs: # 基于文件服务发现
      - files:
        - "./device/nginx_device.yml"   # - "./device/nginx_device.json"  yml和json格式选择一个即可
        refresh_interval: 5s # 每隔5秒检查刷新一次服务发现配置


 vi /usr/local/prometheus/device/nginx_device.yml
 # 或者单独列
- labels:
    desc: nginx     # 无实际意义
  targets:
    - '10.11.7.52:9113'
    - '10.128.2.102:9113'

# 写一起
- labels:
    desc: nginx     # 无实际意义
  targets: ['10.11.7.52:9113','10.128.2.102:9113']


# json模式  file_sd_configs中同步修改
 vi /usr/local/prometheus/device/nginx_device.json
[{
	"labels": {
		"desc": "nginx"
	},
	"targets": [
		"10.11.7.52:9113",
		"10.128.2.102:9113"
	]
}]

······
# 重启Prometheus服务，并访问prometheus所在的target状态，如果为UP说明job设置成功：
[root@prometheus ~]# cd /usr/local/prometheus
[root@prometheus prometheus]# ./promtool check config prometheus.yml
# 保持热加载
[root@prometheus ~]# curl  -X POST localhost:9090/-/reload
```
![](https://qiufuqi.github.io/img/hexo/20231205170451.png)

## Grafana中展示
在 Grafana 中导入 12708 模板 [grafana模板大全参考](https://grafana.com/grafana/dashboards/)
![](https://qiufuqi.github.io/img/hexo/20231205171035.png)
![](https://qiufuqi.github.io/img/hexo/20231205173300.png)
![](https://qiufuqi.github.io/img/hexo/20231205173319.png)

## 编写Nginx告警规则
### 编写记录规则 参考：Nginx-record-rules.yml
```bash
groups:
  - name: Nginx-record
    rules:
    - expr: nginx_up{job=~"Nginx"}
      record: Nginx:up
      labels:
        desc: "节点是否在线, 在线1,不在线0"
        unit: " "
        job: "Nginx"
    - expr: sum(rate(nginx_http_requests_total{job=~"Nginx"}[1m])) by (instance)
      record: Nginx:request:total:rate
      labels:
        desc: "节点的request请求总数"
        unit: "%"
        job: "Nginx"
    - expr: sum(rate(nginx_connections_active{job=~"Nginx"}[1m])) by (instance)
      record: Nginx:request:active:rate
      labels:
        desc: "节点活跃request请求总数"
        unit: "%"
        job: "Nginx"
    - expr: sum(rate(nginx_connections_accepted{job=~"Nginx"}[1m])) by (instance)
      record: Nginx:request:accepted:rate
      labels:
        desc: "节点处理request请求总数"
        unit: "%"
        job: "Nginx"
```
### 编写告警规则 参考：Nginx-alert-rules.yml
```bash
groups:
  - name: Nginx-alert
    rules:
    - alert: Nginx-down
      expr: Nginx:up == 0
      for: 1m
      labels:
        severity: info
      annotations:
        summary: "instance: {{ $labels.instance }} 服务异常"
        description: "{{ $labels.instance }} \n- job: {{ $labels.job }} 服务异常， 时间已经超过1分钟了."
        resolvetion: "{{ $labels.instance }} \n- job: {{ $labels.job }} 服务已恢复."
        value: "{{ $value }}"
        instance: "{{ $labels.instance }}"
        id: "{{ $labels.instanceid }}"
```

# nginx-vts-exporter监控
使用nginx-vts-exporter监控，但是需要在编译nginx的时候添加nginx-module-vts模块，监控数据较多，提供了包含server、upstream以及cache的相关监控指标，指标更加丰富，[部署参考](https://www.cnblogs.com/you-men/p/13173245.html)

## 安装nginx-module-vts
根据nginx版本下载对应的nginx-module-vts，我的nginx是yum安装的，所以需要下载下nginx源码，通过源码编译安装nginx-module-vts
```bash
# 下载源码并解压
cd /opt/ 
wget http://nginx.org/download/nginx-1.24.0.tar.gz
wget https://github.com/vozlt/nginx-module-vts/archive/refs/tags/v0.2.2.tar.gz

tar -zxvf nginx-1.24.0.tar.gz
tar -zxvf v0.2.2.tar.gz && mv nginx-module-vts-0.2.2 nginx-1.24.0

# 备份现有的nginx文件
cp -r /etc/nginx /etc/nginx_bak
cp /usr/sbin/nginx /usr/sbin/nginx_bak

# 查看该源码的nginx支持的模块
cd nginx-1.24.0 && ./configure --help
后面标记disable的，代表已有此模块(编译时,不需要添加）
后面标记enable的，代表不支持此模块(如果有需要,编译时要自己添加该模块）

# 安装依赖
yum -y install perl-ExtUtils-Embed readline-devel zlib-devel pam-devel libxml2-devel libxslt-devel openldap-devel python-devel gcc-c++   openssl-devel cmakepcre-develnanowget  gcc gcc-c++ ncurses-devel per

# 编译 （这里只make，不要make install，不然会覆盖。如果是新装nginx，可以继续make install）
# 编译配置 --add-module=/path/to/nginx-module-vts
# 参考 nginx -V下的configure arguments: 一定要慎重，不然拷贝完就确实很多组件
nginx -V
./configure --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie' --add-module=nginx-module-vts-0.2.2

当出现报错时：./configure: error: no nginx-module-vts-0.2.2/config was found 去掉--with-http_ssl_module重新配置一次

make

# 停止nginx，并做文件替换
systemctl stop nginx
# 文件替换  以上完成后，会在objs目录下生成一个nginx文件
cp objs/nginx /usr/sbin/ 
systemctl start nginx

# 查看nginx是否包含nginx-module-vts-0.2.1 configure arguments: 最后是否有 --add-module=nginx-module-vts-0.2.1
nginx -V

# 删除备份
rm -rf /etc/nginx_bak
rm -rf /usr/sbin/nginx_bak
```
## 添加配置文件
添加相关配置文件，并重启nginx
打开vhost过滤：vhost_traffic_status_filter_by_host on;
开启此功能，在Nginx配置有多个server_name的情况下，会根据不同的server_name进行流量的统计，否则默认会把流量全部计算到第一个
在不想统计流量的server区域禁用vhost_traffic_status
```bash
http {
    vhost_traffic_status_zone;
    vhost_traffic_status_filter_by_host on;

location /status {
        vhost_traffic_status_display;
        vhost_traffic_status_display_format html;
}
```
## 访问展示
浏览器访问http://ip:port/status
![](https://qiufuqi.github.io/img/hexo/20240112162734.png)
## 安装ngxin-vts-exporter
出现以下报错，报错解决
```bash
./nginx-vtx-exporter: /lib64/libc.so.6: version `GLIBC_2.32' not found (required by ./nginx-vtx-exporter)
./nginx-vtx-exporter: /lib64/libc.so.6: version `GLIBC_2.34' not found (required by ./nginx-vtx-exporter)
```
### 脚本安装exporter
nginx-vtx_exporter.sh一键监控安装脚本，提前下载好安装文件或者
```bash
#!/bin/bash
# -*- coding: utf-8 -*-
# Date: 2024/1/15
echo "download nginx-vts-exporter"
sleep 2
wget -N -P /root/ https://github.com/hnlq715/nginx-vts-exporter/releases/download/v0.9.1/nginx-vts-exporter-0.9.1.linux-amd64.tar.gz
 

echo "tar nginx-vts-exporter"
sleep 2
tar -zxvf /root/nginx-vts-exporter-0.9.1.linux-amd64.tar.gz -C /opt/ && mv /opt/nginx-vts-exporter-0.9.1.linux-amd64 /usr/local/nginx-vts-exporter

echo "delete nginx-vts-exporter***tar.gz"
rm -rf /root/nginx-vts-exporter-0.9.1.linux-amd64.tar.gz


echo "firewall nginx-vts-exporter port 9913"
sleep 2
firewall-cmd --zone=public --add-port=9913/tcp --permanent && firewall-cmd --reload 
 
echo "add nginx-vts-exporter.service"
sleep 2
cat << EOF > /usr/lib/systemd/system/nginx-vts-exporter.service
[Unit]
Description=nginx-vts-exporter
After=network.target 
 
[Service]
Restart=on-failure
ExecStart=/usr/local/nginx-vts-exporter/nginx-vts-exporter -nginx.scrape_uri=http://127.0.0.1/status/format/json --telemetry.address=:9913
RestartSec=5
StartLimitInterval=0
StartLimitBurst=10

[Install]
WantedBy=multi-user.target
EOF
 
echo "start nginx-vts-exporter.service"
sleep 2
systemctl daemon-reload && systemctl start nginx-vts-exporter && systemctl enable --now nginx-vts-exporter
```
### 文件安装exporter
[文件下载地址](https://github.com/hnlq715/nginx-vts-exporter/releases)
安装ngxin-vts-exporter，采集机器运行数据信息，默认端口9913 （可更改为指定端口）
```bash
wget https://github.com/hnlq715/nginx-vts-exporter/releases/download/v0.9.1/nginx-vts-exporter-0.9.1.linux-amd64.tar.gz
tar -zxvf /root/nginx-vts-exporter-0.9.1.linux-amd64.tar.gz -C /opt/ && mv /opt/nginx-vts-exporter-0.9.1.linux-amd64 /usr/local/nginx-vts-exporter

#可查看命令 有报错解决报错 解决方案参考
[root@localhost local]# cd /usr/local/nginx-vts-exporter
[root@localhost nginx-vts-exporter]# ./nginx-vts-exporter --help

# 推荐exporter和nginx安装在同一台机器上，如果不在同一台主机，把scrape_uri改为nginx主机的地址。
# nginx_vts_exporter的默认端口号：9913，对外暴露监控接口http://xxx:9913/metrics.
# 我们可以访问浏览器 IP:9913
/usr/local/nginx-vts-exporter/nginx-vts-exporter -nginx.scrape_uri=http://127.0.0.1/status/format/json --telemetry.address=:9913

# 配置node_exporter开机自启
vi /usr/lib/systemd/system/nginx-vts-exporter.service
# 写入以下信息：
[Unit]
Description=nginx-vts-exporter
After=network.target 
 
[Service]
Restart=on-failure
ExecStart=/usr/local/nginx-vts-exporter/nginx-vts-exporter -nginx.scrape_uri=http://127.0.0.1/status/format/json --telemetry.address=:9913
RestartSec=5
StartLimitInterval=0
StartLimitBurst=10

[Install]
WantedBy=multi-user.target

#重启服务
systemctl daemon-reload
systemctl start nginx-vts-exporter
systemctl status nginx-vts-exporter
systemctl enable nginx-vts-exporter
```
## 访问exporter
浏览器访问 IP:9913
![](https://qiufuqi.github.io/img/hexo/20240115120124.png)

## 加入pormetheus
登录prometheus所在服务器，在文件的最下面添加job 配置，并重启Prometheus

### 静态部署
```bash
[root@prometheus ~]# vi /usr/local/prometheus/prometheus.yml
······
  # 静态部署，每次修改需要重启
  - job_name: 'Nginx-vts'
    scrape_interval: 5s
    static_configs:
      - targets: ['10.11.8.108:9913']
······
# 重启Prometheus服务，并访问prometheus所在的target状态，如果为UP说明job设置成功：
[root@prometheus ~]# cd /usr/local/prometheus
[root@prometheus prometheus]# ./promtool check config prometheus.yml
[root@prometheus ~]# systemctl restart prometheus
# 保持热加载
[root@prometheus ~]# curl  -X POST localhost:9090/-/reload
```
### 文件部署
当有新的节点时，只需要修改对应的yml或者json文件即可。
```bash
vi /usr/local/prometheus/prometheus.yml
······
  # 文件部署
  - job_name: 'Nginx-vts'
    scrape_interval: 5s
    file_sd_configs: # 基于文件服务发现
      - files:
        - "./device/nginx-vts_device.yml"   # - "./device/nginx-vts_device.json"  yml和json格式选择一个即可
        refresh_interval: 5s # 每隔5秒检查刷新一次服务发现配置


 vi /usr/local/prometheus/device/nginx-vts_device.yml
 # 或者单独列
- labels:
    desc: nginx     # 无实际意义
  targets:
    - '10.11.8.108:9913'

# 写一起
- labels:
    desc: nginx     # 无实际意义
  targets: ['10.11.8.108:9913']


# json模式  file_sd_configs中同步修改
 vi /usr/local/prometheus/device/nginx-vts_device.json
[{
	"labels": {
		"desc": "nginx"
	},
	"targets": [
		"10.11.8.108:9913",
	]
}]

······
# 重启Prometheus服务，并访问prometheus所在的target状态，如果为UP说明job设置成功：
[root@prometheus ~]# cd /usr/local/prometheus
[root@prometheus prometheus]# ./promtool check config prometheus.yml
# 保持热加载
[root@prometheus ~]# curl  -X POST localhost:9090/-/reload
```
![](https://qiufuqi.github.io/img/hexo/20240115134548.png)

## Grafana中展示
在 Grafana 中导入 2949 模板 [grafana模板大全参考](https://grafana.com/grafana/dashboards/)
![](https://qiufuqi.github.io/img/hexo/20231205171035.png)
![](https://qiufuqi.github.io/img/hexo/20240115144257.png)
![](https://qiufuqi.github.io/img/hexo/20240115144344.png)


| 指标 | 说明  |
| :------------- | :---------- |
|nginx_server_requests	| 统计nginx各个host 各个请求的总数，精确到状态码 |
|nginx_upstream_requests	| 统计各个upstream 请求总数，精确到状态码 |
|nginx_server_connections	| 统计nginx几种连接状态type的连接数 |
|nginx_server_cache	| 统计nginx缓存计算器，精确到每一种状态和转发type |
|nginx_server_bytes	| 统计nginx进出的字节计数可以精确到每个host，in进，out出 |
|nginx_upstream_bytes	| 统计nginx各个 upstream 分组的字节总数，细分到进出 |
|nginx_upstream_responseMsec	| 统计各个upstream 平均响应时长，精确到每个节点 |

