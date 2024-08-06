---
title: Prometheus监控服务发现机制
date: 2023-12-16
tags:
  - Prometheus
  - 钉钉告警
categories: 
- 运维
- 监控
keywords: 'Prometheus,监控,configs'
description: Prometheus监控,告警
cover: https://qiufuqi.github.io/img/hexo/20231205135025.png
abbrlink: prometheus_configs
url: prometheus_configs
comments: false
top: 999
---

Prometheus Server的数据抓取工作于Pull模型，因而，它必需要事先知道各Target的位置，然后才能从相应的Exporter或Instrumentation中抓取数据， 对于小型系统来说，通过static_configs就可以解决此问题，这也是最简单的配置方法；对于中大型系统环境或具有较强动态性的云计算环境来说，静态配置显然难以适用，因此，Prometheus为此专门设计了一组服务发现机制，以便能够通过服务注册中心自动发现、检测、分类可被检测的各target，以及更新发生了变动的target。

## Prometheus指标抓取的生命周期
发现 -> 配置 -> relabel -> 指标数据抓取 -> metrics relabel
[部署参考](https://zhuanlan.zhihu.com/p/656851795)

- 在每个scrape_interval期间，Prometheus都会检查执行的作业（Job）；
- 这些作业首先会根据Job上指定的发现配置生成target列表，此即服务发现过程；
- 服务发现会返回一个Target列表，其中包含一组称为元数据的标签，这些标签都以“__meta_”为前缀；
- 服务发现还会根据目标配置来设置其它标签，这些标签带有“__”前缀和后缀，包括“__scheme__”、 “__address__”和“__metrics_path__”，分别保存有target支持使用协议(http或https，默认为http）、target的地址及指标的URI路径（默认为/metrics）；
- 若URI路径中存在任何参数，则它们的前缀会设置为“__param_；
- 配置标签会在抓取的生命周期中被重复利用以生成其他标签，例如，指标上的instance标签的默认值就来自于__address__标签的值；
- 抓取而来的指标在保存之前，还允许用户对指标重新打标并过滤，在job段metric_relabel_configs配置，通常用来删除不需要的指标、删除敏感或不必要的标签和添加修改标签格式等

### 文件自动发现
此种类型也是最简单的服务发现方式，主要是通过Prometheus Server定期从文件中加载target的信息。文件可以是json或者yaml格式，它含有定义的target列表，以及可选的标签信息。
```bash
vi prometheus.yml
# static config nodes
  - job_name: 'nodes'
    file_sd_configs:
    - files:                                               
      - targets/nodes-*.yaml
      refresh_interval: 2m 
    scrape_interval: 15s
```
将所有要发现的target全部放在targets/目录下即可，例如
```bash
cat targets/nodes-linux.yaml 
- targets:
  - monitor.example.com:9100
  - node.export1.com:9101
  - node.export2.com:9101
  - node.export3.com:9101
  labels:
    app: node-exporter
    os: aliyunos3

cat targets/nodes-prometheus.yaml 
- targets:
  - monitor.example.com:9090
  labels:
    app: prometheus
    job:  prometheus
```

重新加载Prometheus配置即可：
```bash
curl -XPOST monitor.example.com:9090/-/reload
```


## 配置文件参数详解
```bash
# 目标节点 重新打标签 的配置 列表.  重新标记是一个功能强大的工具，可以在抓取目标之前动态重写目标的标签集。 可以配置多个，按照先后顺序应用
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - source_labels: [auth]          # 使用auth 替换param的auth
        target_label: __param_auth
      - source_labels: [module]        # 使用module 替换param的module
        target_label: __param_module
      - target_label: __address__
        replacement: "localhost:9116"  # SNMP Exporter  的地址和端口
```


## 通用配置
通用配置示例参考
```bash
vi prometheus.yml
  - job_name: common
    scrape_interval: 10s
    metrics_path: /metrics
    scheme: http
    file_sd_configs: # 基于文件服务发现
      - files:
        - "./device/common_*.yml"   # - "./device/common_*.json"  yml和json格式选择一个即可
        refresh_interval: 5s
    relabel_configs:
    - source_labels: [job]  
      target_label: job
    - source_labels: [metrics_path]   # 使用metrics_path 替换param的metrics_path
      target_label: __metrics_path__
    - source_labels: [scheme]
      target_label: __scheme__
    - source_labels: [scrape_interval]
      target_label: __scrape_interval__

vi device/common_k8s.yml
- labels:
    job: k8s
    scrape_interval: 10s
    metrics_path: /metrics
  targets:
    - '10.11.7.95:9090'
    - '10.11.7.34:9100'

vi device/common_k8s.yml
[{
	"labels": {
		"job": "k8s",
		"scrape_interval": "10s",
		"metrics_path": "/metrics"
	},
	"targets": [
		"10.11.7.95:9090",
		"10.11.7.34:9100"
	]
}]

```