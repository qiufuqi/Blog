---
title: ARKIME部署安装
date: 2024-8-5
tags:
  - 网络安全
  - arkime
categories: 
- 运维
- 网络安全
- arkime
keywords: '网络安全,arkime'
cover: https://qiufuqi.github.io/img/hexo/20240805144344.png
abbrlink: arkime_install
url: arkime_install
comments: false
---
[文章参考1](https://blog.csdn.net/lhyzws/article/details/124433680) [文章参考2](https://blog.csdn.net/haoyunbin/article/details/131050113)

# 什么是Arkime
Arkime（以前叫Moloch）是一个大规模的开源索引数据包捕获和搜索系统。
Arkime增强了您当前的安全基础设施，以标准PCAP格式存储和索引网络流量，提供快速的索引访问。为PCAP浏览、搜索和导出提供了直观简单的web界面。Arkime公开了API，允许直接下载和使用PCAP数据和JSON格式的会话数据。Arkime以标准PCAP格式存储和导出所有数据包，允许您在分析工作流程中使用最喜欢的PCAP摄取工具，如wireshark。
Arkime可以跨多个系统部署，可以扩展到处理每秒数十GB的流量。PCAP保留基于可用的传感器磁盘空间。元数据保留基于Elasticsearch集群规模。两者都可以随时增加，并在您的完全控制下。

# 下载和安装
[下载arkime安装包](https://github.com/arkime/arkime/releases/)
根据系统版本下载对应版本，对于Centos 8或RHEL 8使用el8下载，而不是RHEL 9。本次服务器为centos 8，所以使用arkime-main.el8.x86_64.rpm包
![](https://qiufuqi.github.io/img/hexo/20240806153208.png)

## 安装依赖
arkime安装主要就3个依赖，分别是perl-JSON、perl-libwww-perl和perl-LWP-Protocol-https。使用yumdownloader解析并下载依赖包后安装。
![](https://qiufuqi.github.io/img/hexo/20240806194219.png)
```bash
yumdownloader --resolve perl-JSON perl-LWP-Protocol-https perl-libwww-perl
yum -y install perl-JSON perl-LWP-Protocol-https perl-libwww-perl
```
## 安装arkime
执行以下命令安装arkime，**rpm -ivh arkime-main.el8.x86_64.rpm**
``` bash
[root@LYGVLTOPO03 ~]# rpm -ivh arkime-main.el8.x86_64.rpm 
Verifying...                          ################################# [100%]
准备中...                          ################################# [100%]
正在升级/安装...
   1:arkime-5.4.0_GIT-10201223847     ################################# [100%]
Arkime systemd files copied
Installing logrotate /etc/logrotate.d/arkime to delete files after 14 days
READ /opt/arkime/README.txt and RUN /opt/arkime/bin/Configure

```
![](https://qiufuqi.github.io/img/hexo/20240806194557.png)
## 安装elastic
如何安装elasticsearch，如何配置arkime，在 **/opt/arkime/bin/Configure** 这个脚本中列得很清楚。
打开该脚本，找到如下安装ES的这一行（大约203行），手动下载该rpm并安装，或 根据ES_VERSION手动下载该[elasticsearch](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-oss-7.10.2-x86_64.rpm),并执行安装
``` bash
[root@LYGVLTOPO03 ~]# vi /opt/arkime/bin/Configure
·········
if [ "$ARKIME_INSTALLELASTICSEARCH" == "yes" ]; then
    echo "Arkime - Downloading and installing demo OSS version of Elasticsearch"
    ES_VERSION=7.10.2
    if [ -f "/etc/redhat-release" ] || [ -f "/etc/system-release" ]; then
# 新增下面一行 wget
        wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-oss-${ES_VERSION}-${ARCHRPM}.rpm
        yum install https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-oss-${ES_VERSION}-${ARCHRPM}.rpm
    else
        wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-oss-${ES_VERSION}-${ARCHDEB}.deb
        dpkg -i elasticsearch-oss-${ES_VERSION}-$ARCHDEB.deb
        /bin/rm -f elasticsearch-oss-${ES_VERSION}-$ARCHDEB.deb
    fi
fi
·········
```
执行安装命令 **rpm -ivh elasticsearch-oss-7.10.2-x86_64.rpm** ,并设置开机自启动
``` bash
[root@LYGVLTOPO03 ~]# rpm -ivh elasticsearch-oss-7.10.2-x86_64.rpm

# 安装成功后 设置开机自启动
[root@LYGVLTOPO03 ~]# systemctl daemon-reload
[root@LYGVLTOPO03 ~]# systemctl enable elasticsearch.service
[root@LYGVLTOPO03 ~]# systemctl start elasticsearch.service
```
## 配置arkime
首先确认好该服务器网卡名称 **ip addr** 比如该服务器为：ens192
安装好后执行 **/opt/arkime/bin/Configure** 配置工具进行后续配置，es已经提前安装，所以选择no （如果没有选择yes），geo数据库建议选no。
依次输入 ens192 no arkime no 
``` bash
[root@LYGVLTOPO03 ~]# /opt/arkime/bin/Configure 
Found interfaces: ens192;lo
Semicolon ';' seperated list of interfaces to monitor [eth1] ens192
Install Elasticsearch server locally for demo, must have at least 3G of memory, NOT recommended for production use (yes or no) [no] no
OpenSearch/Elasticsearch server URL [https://localhost:9200] 
OpenSearch/Elasticsearch user [empty is no user] 
Password to encrypt S2S and other things, don't use spaces [must create one] 
Password to encrypt S2S and other things, don't use spaces [must create one] arkime
Arkime - Creating configuration files
Not overwriting /opt/arkime/etc/config.ini, delete and run again if update required (usually not), or edit by hand
Download GEO files? You'll need a MaxMind account https://arkime.com/faq#maxmind (yes or no) [yes] no
Arkime - NOT downloading GEO files

Arkime - Configured - Now continue with step 4 in /opt/arkime/README.txt

 4) The Configure script can install OpenSearch/Elasticsearch for you or you can install yourself
 5) Initialize/Upgrade OpenSearch/Elasticsearch Arkime configuration
  a) If this is the first install, or want to delete all data
      /opt/arkime/db/db.pl http://ESHOST:9200 init
  b) If this is an update to an Arkime package
      /opt/arkime/db/db.pl http://ESHOST:9200 upgrade
 6) Add an admin user if a new install or after an init
      /opt/arkime/bin/arkime_add_user.sh admin "Admin User" THEPASSWORD --admin
 7) Start everything
      systemctl start arkimecapture.service
      systemctl start arkimeviewer.service
 8) Look at log files for errors
      /opt/arkime/logs/viewer.log
      /opt/arkime/logs/capture.log
 9) Visit http://arkimeHOST:8005 with your favorite browser.
      user: admin
      password: THEPASSWORD from step #6

If you want IP -> Geo/ASN to work, you need to setup a maxmind account and the geoipupdate program.
See https://arkime.com/faq#maxmind

Any configuration changes can be made to /opt/arkime/etc/config.ini
See https://arkime.com/faq#arkime-is-not-working for issues

Additional information can be found at:
  * https://arkime.com/install
  * https://arkime.com/faq
  * https://arkime.com/settings
```
![](https://qiufuqi.github.io/img/hexo/20240806201737.png)
参考屏幕上返回的指示完成剩下的4、5、6、7、8、9步

在启动elasticsearch服务后，需要看看ES是否正常——有时虽然服务是开的，端口是开的，但是执行curl **http://localhost:9200** 时就是看不到如下的结果，那么再继续往下就是没有意义的
``` bash
# 初始化ES
[root@LYGVLTOPO03 ~]# /opt/arkime/db/db.pl http://localhost:9200 init
[root@LYGVLTOPO03 ~]# curl http://localhost:9200
{
  "name" : "LYGVLTOPO03",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "hCgB6VGiRyO1Zj43h4eslw",
  "version" : {
    "number" : "7.10.2",
    "build_flavor" : "oss",
    "build_type" : "rpm",
    "build_hash" : "747e1cc71def077253878a59143c1f785afa92b9",
    "build_date" : "2021-01-13T00:42:12.435326Z",
    "build_snapshot" : false,
    "lucene_version" : "8.7.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```
``` bash
# 增加用户名和密码，用来登录Arkime界面
[root@LYGVLTOPO03 ~]# /opt/arkime/bin/arkime_add_user.sh admin "Admin User" arkime --admin
# 这一步可能报错，根据报错信息排查
Common issues:
  * Is OpenSearch/Elasticsearch running and NOT RED?
  * Have you run 'db/db.pl <host:port> init'?
  * Is the 'elasticsearch' setting (https://localhost:9200) correct in config file (/opt/arkime/etc/config.ini) with a username and password if needed? (https://arkime.com/settings#elasticsearch)
  * Do you need the --insecure option? (See https://arkime.com/faq#insecure)

# 锁定第三步 修改
elasticsearch=https://localhost:9200
elasticsearchBasicAuth=admin:arkime

elasticsearch=http://localhost:9200
#elasticsearchBasicAuth=
```
启动arkime
``` bash
[root@LYGVLTOPO03 ~]# systemctl start arkimecapture.service
[root@LYGVLTOPO03 ~]# systemctl start arkimeviewer.service
[root@LYGVLTOPO03 ~]# systemctl enable arkimecapture.service
[root@LYGVLTOPO03 ~]# systemctl enable arkimeviewer.service
```
浏览器访问http://10.50.2.251:8005/ 输入账号admin，密码，此时数据是空的
![](https://qiufuqi.github.io/img/hexo/20240806203649.png)

## 采集数据
在Configure脚本，能跟踪到此处是执行 **/opt/arkime/bin/arkime_update_geo.sh**，主要目的就是下载2个文件。
执行该脚本或者从arkime_update_geo.sh直接拿出两个地址进行wget 至/opt/arkime/etc目录下，第二个地址下载下来的文件应该改名为oui.txt
https://www.iana.org/assignments/ipv4-address-space/ipv4-address-space.csv
https://www.wireshark.org/download/automated/data/manuf
``` bash
[root@LYGVLTOPO03 bin]# sh arkime_update_geo.sh 
2024-08-06 20:40:02 URL:https://www.iana.org/assignments/ipv4-address-space/ipv4-address-space.csv [23323/23323] -> "/tmp/tmp.wzDlgTvE31" [1]
2024-08-06 20:40:07 URL:https://www.wireshark.org/download/automated/data/manuf [2802619/2802619] -> "/tmp/tmp.orTlPb8fL6" [1]

root@LYGVLTOPO03 bin]# cd ../etc
[root@LYGVLTOPO03 etc]# ls
arkimecapture.systemd.service     arkimeviewer.systemd.service  config.ini.sample   ipv4-address-space.csv  parliament.ini.sample
arkimecont3xt.systemd.service     arkimewise.systemd.service    cont3xt.ini.sample  oui.txt                 wise.ini.sample
arkimeparliament.systemd.service  config.ini                    env.example         parliament.env.example
[root@LYGVLTOPO03 etc]# 

# 下载完成后重启capture服务
[root@LYGVLTOPO03 ~]# systemctl restart arkimecapture.service
```
过一会就会有数据了
![](https://qiufuqi.github.io/img/hexo/20240806204520.png)
# 高性能配置
``` bash
# 修改arkime配置文件/opt/arkime/etc/config.ini 启用如下参数
[root@LYGVLTOPO03 ~]# vi /opt/arkime/etc/config.ini
magicMode=basic
pcapReadMethod=tpacketv3
tpacketv3NumThreads=2
pcapWriteMethod=simple
pcapWriteSize=2560000
packetThreads=5
maxPacketsInQueue=200000

注：修改配置文件后，要重启arkime服务
systemctl restart arkimecapture
```