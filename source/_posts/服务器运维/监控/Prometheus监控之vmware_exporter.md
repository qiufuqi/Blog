---
title: Prometheus监控之vmware_exporter
date: 2024-1-11
tags:
  - Prometheus
  - vmware
categories: 
- 运维
- 监控
- vmware
keywords: 'Prometheus,中间件,vmware'
description: Prometheus监控,中间件
cover: https://qiufuqi.github.io/img/hexo/20240111084553.png
abbrlink: prometheus_vmware
url: prometheus_vmware
comments: false
top: 995
---

[部署参考](https://blog.csdn.net/hanzheng260561728/article/details/128128498)

## 部署docker
提前安装[部署docker环境](/docker_install)

## 防火墙
放行9272端口或者关闭防火墙
```bash
firewall-cmd --zone=public --add-port=9272/tcp --permanent
firewall-cmd --reload
```
## 编写配置文件
docker 运行vmware_exporter，端口9272
```bash
docker pull pryorda/vmware_exporter

# 测试运行
docker run -it --rm  -p 9272:9272 -e VSPHERE_USER=${VSPHERE_USERNAME} -e VSPHERE_PASSWORD=${VSPHERE_PASSWORD} -e VSPHERE_HOST=${VSPHERE_HOST} -e VSPHERE_IGNORE_SSL=True -e VSPHERE_SPECS_SIZE=2000 --name vmware_exporter pryorda/vmware_exporter

docker run -it --rm  -p 9272:9272 --env-file /usr/local/vmware_exporter/config.env --name vmware_exporter pryorda/vmware_exporter




# 正式运行
docker run -d --restart="always" -p 9272:9272 --name vmware_exporter  --env-file /usr/local/vmware_exporter/config.env pryorda/vmware_exporter

cat /usr/local/vmware_exporter/config.env
VSPHERE_USER=administrator@vsphere.local
VSPHERE_PASSWORD=Password
VSPHERE_HOST=10.11.7.24
VSPHERE_IGNORE_SSL=TRUE
VSPHERE_SPECS_SIZE=2000
```
## 采集结果查看
浏览器访问http://YOUIP:9272/metrics
![](https://qiufuqi.github.io/img/hexo/20240111104201.png)

## 加入prometheus
[配置参考](https://github.com/pryorda/vmware_exporter)
```bash
  - job_name: 'vmware_esx'
    metrics_path: '/metrics'
    file_sd_configs:
      - files:
        - /etc/prometheus/esx.yml
    params:
      section: [esx]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9272

  - job_name: vmware_export
    metrics_path: /metrics
    static_configs:
    - targets:
      - vcenter01
      - vcenter02
    relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: exporter_ip:9272

  - job_name: vmware_export
    scrape_interval: 5s
    scrape_timeout: 10s
    metrics_path: /metrics
    static_configs:
    - targets:
      - 10.11.7.24
    relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: 10.11.8.108:9272

```
## Grafana中展示
在 Grafana 中导入 11243 模板 [grafana模板大全参考](https://grafana.com/grafana/dashboards/)
![](https://qiufuqi.github.io/img/hexo/20231205171035.png)
![](https://qiufuqi.github.io/img/hexo/20240111111441.png)

## 告警规则
```bash
groups:
    - name: EXSi主机状态监控告警
      rules:
      - alert: EXSi主机状态
        expr: vmware_host_power_state ==0
        for: 5m
        labels:
          type: lost
          severity: fatal
        annotations:
          summary: "EXSi主机 {{$labels.host_name}} 失联"
          description: "EXSi任务 {{$labels.job}} 下的主机 {{$labels.host_name}} 已经超过五分钟没有数据了."
          monitor_url: "http://10.0.10.120:3000/d/q1yCDNbWz/vmware-stats?orgId=1"
 
      - alert: EXSi主机CPU使用情况
        expr: (vmware_host_cpu_usage / vmware_host_cpu_max) * 100 >80
        for: 5m
        labels:
          type: cpu
          severity: warning
        annotations:
          summary: "EXSi主机 {{ $labels.host_name }} 的 CPU 使用率告警"
          description: "EXSi主机 {{ $labels.host_name }} CPU 使用率超过 80%, 当前值为： {{ $value }}"
          monitor_url: "http://10.0.10.120:3000/d/q1yCDNbWz/vmware-stats?orgId=1"
 
      - alert: EXSi主机内存使用
        expr: (vmware_host_memory_usage/ vmware_host_memory_max) * 100 >85
        for: 5m
        labels:
          type: mem
          severity: warning
        annotations:
          summary: "EXSi主机 {{ $labels.host_name }} 的内存使用率告警"
          description: "EXSi主机 {{ $labels.host_name }} 的内存使用率超过 85%, 当前值为： {{ $value }}"
          monitor_url: "http://10.0.10.120:3000/d/q1yCDNbWz/vmware-stats?orgId=1"
 
      - alert: EXSi主机磁盘容量
        expr: ((vmware_datastore_capacity_size- vmware_datastore_freespace_size) / vmware_datastore_capacity_size) * 100  >70
        for: 5m
        labels:
          type: cpu
          severity: warning
        annotations:
          summary: "EXSi主机 {{ $labels.host_name }} 的磁盘使用率告警"
          description: "EXSi主机 {{ $labels.host_name }} 的磁盘使用率超过 70%, 挂载点: {{ $labels.mountpoint }} 当前值为：{{ $value }}%"
          monitor_url: "http://10.0.10.120:3000/d/q1yCDNbWz/vmware-stats?orgId=1"
————————————————
版权声明：本文为CSDN博主「遥襟」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/weixin_46396833/article/details/118021606
```




