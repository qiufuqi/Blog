---
title: Prometheus监控之snmp_exporter
date: 2023-12-07
tags:
  - Prometheus
  - snmp
categories: 
- 运维
- 监控
- snmp
keywords: 'Prometheus,中间件,snmp'
description: Prometheus监控,中间件
cover: https://qiufuqi.github.io/img/hexo/20231213164836.png
abbrlink: prometheus_snmp
url: prometheus_snmp
comments: false
top: 996
---

SNMP Exporter是一个用于监控和收集网络设备信息的开源软件。它作为Prometheus监控系统的一个重要组件，通过使用Simple Network Management Protocol（SNMP）协议，从网络设备（如路由器、交换机、服务器等）中提取性能指标和状态信息，并将其转换为Prometheus可以理解的格式，从而实现对网络设备的监控和指标收集。

对于 SNMP Exporter 的使用来说， 配置文件比较重要，配置文件中根据硬件的 MIB 文件生成了 OID 的映射关系。以 Cisco 交换机为例，在官方 GitHub 上下载最新的 snmp.yml 文件，由于 Cisco 交换机使用的是 if_mib 模块，在 if_mib 下新增 auth ，用来在请求交换机的时候做验证使用，这个值是配置在交换机上的。
[部署参考](https://blog.csdn.net/hanzheng260561728/article/details/127923009)

## 安装snmp_exporter
### 脚本安装snmp_exporter
snmp_exporter.sh一键监控安装脚本，提前下载好安装文件或者[在线下载](https://github.com/prometheus/snmp_exporter/releases)
```bash
#!/bin/bash
# -*- coding: utf-8 -*-
# Date: 2023/12/13
echo "download snmp_exporter"
sleep 2
wget -N -P /root/ https://github.com/prometheus/snmp_exporter/releases/download/v0.25.0/snmp_exporter-0.25.0.linux-amd64.tar.gz

echo "tar snmp_exporter"
sleep 2
tar -zxvf /root/snmp_exporter-0.25.0.linux-amd64.tar.gz -C /opt/ && mv /opt/snmp_exporter-0.25.0.linux-amd64 /usr/local/snmp_exporter

echo "delete snmp_exporter***tar.gz"
rm -rf /root/snmp_exporter-0.25.0.linux-amd64.tar.gz

echo "firewall snmp_exporter port 9116"
sleep 2
firewall-cmd --zone=public --add-port=9116/tcp --permanent && firewall-cmd --reload 
 
echo "add snmp_exporter.service"
sleep 2
cat << EOF > /usr/lib/systemd/system/snmp_exporter.service
[Unit]
Description=snmp_exporter
After=network.target 
 
[Service]
Restart=on-failure
ExecStart=/usr/local/snmp_exporter/snmp_exporter --config.file=/usr/local/snmp_exporter/snmp.yml --web.listen-address=:9116 --snmp.wrap-large-counters --log.level=info

[Install]
WantedBy=multi-user.target
EOF
 
echo "start snmp_exporter.service"
sleep 2
systemctl daemon-reload && systemctl start snmp_exporter && systemctl enable --now snmp_exporter
```

### 文件安装snmp_exporter
[文件下载地址](https://github.com/prometheus/snmp_exporter/releases)
安装snmp_exporter，采集机器运行数据信息，默认端口9116 （可更改为指定端口）
```bash
wget -N -P /root/ https://github.com/prometheus/snmp_exporter/releases/download/v0.25.0/snmp_exporter-0.25.0.linux-amd64.tar.gz

#解压至指定文件夹
tar -zxvf /root/snmp_exporter-0.25.0.linux-amd64.tar.gz -C /opt/ 
mv /opt/snmp_exporter-0.25.0.linux-amd64 /usr/local/snmp_exporter

#可查看命令
[root@localhost ~]# cd /usr/local/snmp_exporter
[root@localhost snmp_exporter]# ./snmp_exporter  -h

# 下述命令中，--nginx.scrape-uri参数指定了收集指标信息的URI地址，此处的地址是Nginx的状态页。
[root@localhost snmp_exporter]# /usr/local/snmp_exporter/snmp_exporter --config.file=/usr/local/snmp_exporter/snmp.yml --web.listen-address=:9116 --snmp.wrap-large-counters --log.level=info

# 配置node_exporter开机自启
vi /usr/lib/systemd/system/snmp_exporter.service
# 写入以下信息：
[Unit]
Description=snmp_exporter
After=network.target 
 
[Service]
Restart=on-failure
ExecStart=/usr/local/snmp_exporter/snmp_exporter --config.file=/usr/local/snmp_exporter/snmp.yml --web.listen-address=:9116 --snmp.wrap-large-counters --log.level=info

[Install]
WantedBy=multi-user.target

#重启服务
systemctl daemon-reload
systemctl start snmp_exporter
systemctl status snmp_exporter
systemctl enable snmp_exporter
```

## 配置snmp_exporter
提前确定好snmp的团体字，做好验证工作，修改snmp.yml文件，将community值改为对应团体字，重启服务，并测试采集情况
此处不同版本有不同写法，[参考建议](https://github.com/prometheus/snmp_exporter/blob/main/auth-split-migration.md)

验证采集情况: curl 'http://YOU_snmp_exporter_IP:9116/snmp?module=if_mib&target=YOU_SW_IP'
- http://YOU_snmp_exporter_ip:9116
- Target #是交换机IP
- Module #是你的snmp.yml 配置文件内部定义的名称

默认使用的public_v2验证信息，当有不同团体字时，需要新生成接口或者增加
```bash
[root@localhost ~]# vi /usr/local/snmp_exporter/snmp.yml
# WARNING: This file was auto-generated using snmp_exporter generator, manual changes will be lost.
auths:
  public_v1:
    community: cisco45
    security_level: noAuthNoPriv
    auth_protocol: MD5
    priv_protocol: DES
    version: 1
  public_v2:
    community: cisco45
    security_level: noAuthNoPriv
    auth_protocol: MD5
    priv_protocol: DES
    version: 2
  public_v3:
    community: admin@kk
    security_level: noAuthNoPriv
    auth_protocol: MD5
    priv_protocol: DES
    version: 2
·········
[root@localhost ~]# systemctl restart snmp_exporter
[root@localhost ~]# curl 'http://10.11.8.108:9116/snmp?module=if_mib&target=10.10.10.140'
```

## 访问snmp_exporter
浏览器运行访问 http://10.11.8.108:9116/
![](https://qiufuqi.github.io/img/hexo/20231214143537.png)
![](https://qiufuqi.github.io/img/hexo/20231215095224.png)


## 生成snmp.yml文件
本步骤可不用做，使用常用的配置即可，不同厂商可能mid不同，[参考](http://localnetwork.cn/project-10/doc-299/)
snmp_exporter的配置文件 snmp.yml 需要自己通过SNMP Exporter Config Generator 项目编译生成，由于Prometheus使用go语言开发的，所以自己编译生成snmp_exporter的配置文件需要go环境
### go环境安装
直接yum安装
```bash
rpm --import https://mirror.go-repo.io/centos/RPM-GPG-KEY-GO-REPO
curl -s https://mirror.go-repo.io/centos/go-repo.repo | tee /etc/yum.repos.d/go-repo.repo
yum -y install golang
[root@localhost ~]# go version
go version go1.21.5 linux/amd64
```
源码安装
```bash
wget https://dl.google.com/go/go1.21.5.linux-amd64.tar.gz
tar -zxvf go1.21.5.linux-amd64.tar.gz -C /usr/local/

vi /etc/profile
# 在最后一行添加
export GOROOT=/usr/local/go
export PATH=$PATH:$GOROOT/bin

[root@localhost ~]# source /etc/profile && go version
go version go1.21.5 linux/amd64
```

### snmp Generator安装
go环境安装以后，构建snmp exporter  config Generator，执行以下操作: [操作参考](https://github.com/prometheus/snmp_exporter/tree/main/generator)
```bash
yum -y install git gcc gcc-g++ make net-snmp net-snmp-utils net-snmp-libs net-snmp-devel

# 可能会报错 安装解决需要：libmysqlclient.so.18 问题 需要和数据库一致版本
wget https://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/mysql-community-libs-compat-5.7.31-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-compat-5.7.31-1.el7.x86_64.rpm
yum -y install git gcc gcc-g++ make net-snmp net-snmp-utils net-snmp-libs net-snmp-devel
```

### 编译snmp.yml
curl版本低时会报错，此时需要升级curl，[curl升级参考](/centos_curl)
```bash
cd /opt/
git clone https://github.com/prometheus/snmp_exporter.git
go env -w GO111MODULE=on                        #GO111MODULE开启
go env -w GOPROXY=https://goproxy.cn,direct     #选择国内代理
cd /opt/snmp_exporter/generator
make generator mibs                             ## mibs文件夹中放入对应品牌的无线设备mib库文件即可


# 以上命令可能报错
curl: option --no-progress-meter: is unknown    # curl需要升级

# 国内网络下载开源项目提供的公共MIB库报错 忽略即可或者科学上网
# curl: (28) Failed to connect to www.circitor.fr port 443 after 127263 ms
```
修改配置文件，增加community: cisco45，不同团体子可新增v3,v4,v5等等，但是版本都用version:2。这样生成snmp.yml里会自动带上
```bash
vi generator.yml
---
auths:
  public_v1:
    version: 1
  public_v2:
    version: 2
    community: cisco45
  public_v3:
    version: 2
    community: admin@yr

#所需要监控的项添加进去



export MIBDIRS=/opt/snmp_exporter/generator/mibs
./generator generate           #编译成功后，会自动生成snmp.yml文件
cp snmp.yml /usr/local/snmp_exporter/snmp.yml   #将snmp.yml文件传到snmp_exporter安装目录下
systemctl  restart snmp_exporter                #重启该服务
```
## 加入prometheus监控
登录prometheus所在服务器，在文件的最下面添加job 配置，并重启Prometheus
```bash
[root@prometheus ~]# vi /usr/local/prometheus/prometheus.yml
······
# 参考：
  - job_name: "SNMP"
    scrape_interval: 1m  # 默认抓取周期，可用单位m s、smhdwy #全局默认1分钟
    scrape_timeout: 30s  # 默认抓取超时，全局默认10s
    static_configs:
      - targets:
          - 10.10.10.140  # 思科交换机的 IP 地址
    metrics_path: /snmp
    params:
      module:
        - if_mib  # 如果是其他设备，请更换其他模块。
      auth: 
        - public_v1     # 认证使用/usr/local/snmp_exporter/snmp.yml的认证名 包含团体字
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: "localhost:9116"  # SNMP Exporter  的地址和端口

# 最终使用以下配置
  - job_name: 'Snmp'
    scrape_interval: 30s    # 默认抓取周期，可用单位m s、smhdwy #全局默认1分钟
    scrape_timeout: 30s     # 默认抓取超时，全局默认10s
    static_configs:
      - targets: ['10.10.10.140','10.10.10.141']
    metrics_path: /snmp
    params:
      module: [if_mib]
      auth: 
        - public_v1
    relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: 10.11.8.108:9116
······
# 重启Prometheus服务，并访问prometheus所在的target状态，如果为UP说明job设置成功：
[root@prometheus ~]# cd /usr/local/prometheus
[root@prometheus prometheus]# ./promtool check config prometheus.yml
[root@prometheus ~]# systemctl restart prometheus
# 保持热加载
[root@prometheus ~]# curl  -X POST localhost:9090/-/reload
```
![](https://qiufuqi.github.io/img/hexo/20231215103201.png)


## 多机器采集
多机器不同模块采集，基于文件服务发现。一般情况下，交换机都是有多台，甚至几百上千台，在如此多的设备需要监控采集数据，需要指定不同模块和不同配置文件进行加载采集的，下面简单介绍下多机器部署采集。[部署参考](http://localnetwork.cn/project-10/doc-299/)

Prometheus可以根据不同的文件格式（JSON或YAML）和服务描述信息自动发现并监控服务。这对于动态环境或需要自动扩展的服务非常有用，因为当服务发生变化时，Prometheus可以自动更新其监控配置并开始监控新的服务
file_sd_configs 发现服务 用json或者yaml格式文件方式，都可以

针对不同团体字，snmp_exporter生成新的auth认证方式，需要修改relabel_configs，参考黑框处自己编写 ，用Labels里面的值替换参数值
```bash
    - source_labels: [auth]        # 使用module 替换param的module
      target_label: __param_auth
    - source_labels: [module]      # 使用module 替换param的module
      target_label: __param_module
```
![](https://qiufuqi.github.io/img/hexo/20231215165747.png)

```bash
[root@prometheus ~]# vi /usr/local/prometheus/prometheus.yml
·········
# 参考
  - job_name: "SNMP"
    scrape_interval: 1m # 覆盖全局默认值
    scrape_timeout: 30s # 覆盖全局默认值
    file_sd_configs: # 基于文件服务发现
      - files:
        - "./device/snmp_device.yml" # 指定 snmp 服务发现配置文件路径
        refresh_interval: 5s # 每隔5秒检查刷新一次服务发现配置
    metrics_path: /snmp
    params:       # 如果snmp_device.yml里面写了params的参数，则删除此处
      module:
        - if_mib  # 指定默认采集 MIB 模块的名称,如果是其他设备，请更换其他模块。
      auth: 
        - public_v2     # 认证使用snmp.yml的某个认证名 包含团体字
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
snmp_device.yml 的内容参照如下格式即可。我在下面的示例中添加了architecture与model等变量，这些变量prometheus获取目标信息是，会作为目标的标签与目标绑定。涉及不同团体字时，snmp中的auth需要提前生成才可以使用，比如我的 public_v3
```bash
# 部署参考
vi /url/local/prometheus/device/snmp_device.yml
- labels:
    mib: HZHUAWEI  # 这里的名字只是个标签
    hostname: HZ-NL-HW5720STACK
  targets:
    - 192.168.100.1
- labels:
    mib: if_mib
    hostname: HZ-HX-HW12704CSS
  targets:
    - 10.10.10.140
- labels:
    mib: if_mib
  targets:
    - 10.10.10.141

# 最终版本
- labels:
    module: if_mib
    auth: public_v2
  targets: ['10.10.10.140','10.10.10.141']
- labels:
    module: if_mib
    auth: public_v3
  targets: ['10.10.10.31']
```
![](https://qiufuqi.github.io/img/hexo/20231216103813.png)


使用json文件格式时
```bash
vi /usr/local/prometheus/prometheus.yml
    file_sd_configs: # 基于文件服务发现
      - files:
        - "./device/snmp_device.json" # 指定 snmp 服务发现配置文件路径
        refresh_interval: 5s # 每隔5秒检查刷新一次服务发现配置


vi /usr/local/prometheus/device/snmp_device.json
[{
		"labels": {
			"module": "if_mib",
			"auth": "public_v2"
		},
		"targets": ["10.10.10.140", "10.10.10.141"]
	},
	{
		"labels": {
			"module": "if_mib",
			"auth": "public_v3"
		},
		"targets": ["10.10.10.31"]
	}
]

curl -XPOST localhost:9090/-/reload
```


## Grafana中展示
在 Grafana 中导入 15473 模板 [grafana模板大全参考](https://grafana.com/grafana/dashboards/)
![](https://qiufuqi.github.io/img/hexo/20231205171035.png)
![](https://qiufuqi.github.io/img/hexo/20231215104944.png)
![](https://qiufuqi.github.io/img/hexo/20231215105016.png)


在 Grafana 中导入 11169 模板 [grafana模板大全参考](https://grafana.com/grafana/dashboards/)
![](https://qiufuqi.github.io/img/hexo/20231215105518.png)


## 编写SNMP告警规则
未完待续
