---
title: oxidized网络设备自动备份
date: 2024-8-8
tags:
  - 网络安全
  - oxidized
categories: 
- 运维
- 网络安全
- oxidized
keywords: '网络安全,oxidized'
cover: https://qiufuqi.github.io/img/hexo/20240812085958.png
abbrlink: oxidized_install
url: oxidized_install
comments: false
---
[文字参考](https://cloud.tencent.com/developer/article/1654423) [Docker部署参考](https://www.cnblogs.com/alanlin/p/11173214.html)
[项目地址](https://github.com/ytti/oxidized)
随着网络设备的增多，通过人手备份网络设备倍感压力，而且效率低。有编程基础的人可能会通过Python的parimiko 或者netmiko 连接到设备操作 把文件通过ftp 上传到FTP服务器, 在通过定时任务,定期自动备份。这个应该是现阶段主流非人民币网络玩家的最优解决方案。
今天我们来看看oxidized这个被称之为“最好用的”网络备份系统，友好的支持不同厂商。

# oxidized简介
oxidized 是一个网络设备备份系统, 轻量级,可扩展,支持超过90多个操作系统。个人觉得它无与伦比的优势, 同时支持h3c,华为,思科。
随着容器化的兴起，部署软件变得的越来越简单，有的已经帮您封装好，你开箱即用就可以了。
oxidized组成：
- config 文件：oxidized 配置文件
- Sources 字段：定位 router.db 文件的位置
- Outputs 字段 ：设备备份文件的存储位置
- model 字段：设备厂商所用的系统, 核心功能就是靠这个实现的
- router.db文件：被管网络设备详细信息

# 部署安装
## 拉取官方镜像
把官方的 oxidized/oxidized 镜像拉下来，有可能会失败，则自己制作镜像
``` bash
[root@LYGVLTOPOT01 ~]# docker pull repository-tst.hengrui.com/docker-proxy/oxidized/oxidized:latest
latest: Pulling from oxidized/oxidized
toomanyrequests: You have reached your pull rate limit. You may increase the limit by authenticating and upgrading: https://www.docker.com/increase-rate-limit
```
## 制作镜像
如果上一步拉取失败，则自己制作镜像
``` bash
# clone oxidized文件
git clone https://github.com/ytti/oxidized
# build xoidized镜像 -- 可能会有报错
docker build -q -t oxidized/oxidized:latest oxidized/
```
## 启动镜像
``` bash
docker run --name oxidized -it -v /etc/localtime:/etc/localtime:ro -v /data/oxidized:/home/oxidized/.config/oxidized -p 8888:8888/tcp -e CONFIG_RELOAD_INTERVAL=3600 --restart unless-stopped -d oxidized/oxidized 

# 查看日志 && 进入oxidized内部
docker logs -n 50 oxidized
docker exec -it oxidized /bin/bash
```
发现首次会启动失败，提示缺少router.db文件
``` bash
[root@LYGVLTOPOT01 oxidized]# docker logs -n 50 oxidized 
*** Running /etc/my_init.d/00_regen_ssh_host_keys.sh...
*** Running /etc/my_init.d/10_syslog-ng.init...
Aug 12 15:27:45 3d76614d6801 syslog-ng[14]: syslog-ng starting up; version='3.35.1'
*** Booting runit daemon...
*** Runit started as PID 24
Aug 12 15:27:46 3d76614d6801 cron[30]: (CRON) INFO (pidfile fd = 3)
Aug 12 15:27:46 3d76614d6801 cron[30]: (CRON) INFO (Running @reboot jobs)
edit ~/.config/oxidized/config
I, [2024-08-12T15:27:48.453778 #40]  INFO -- : Oxidized starting, running as pid 40
F, [2024-08-12T15:27:48.456570 #40] FATAL -- : Oxidized crashed, crashfile written in /home/oxidized/.config/oxidized/crash
no source csv config, edit ~/.config/oxidized/config
I, [2024-08-12T15:27:49.751378 #42]  INFO -- : Oxidized starting, running as pid 42
I, [2024-08-12T15:27:49.760121 #42]  INFO -- : lib/oxidized/nodes.rb: Loading nodes
F, [2024-08-12T15:27:49.955537 #42] FATAL -- : Oxidized crashed, crashfile written in /home/oxidized/.config/oxidized/crash
no output file config, edit ~/.config/oxidized/config
I, [2024-08-12T15:27:51.283274 #44]  INFO -- : Oxidized starting, running as pid 44
I, [2024-08-12T15:27:51.284005 #44]  INFO -- : lib/oxidized/nodes.rb: Loading nodes
I, [2024-08-12T15:27:51.481188 #44]  INFO -- : lib/oxidized/nodes.rb: Loaded 1 nodes
```
## 修改配置
此处的顺序和config中map调用有关系
``` bash
[root@LYGVLTOPOT01 ~]# cat /data/oxidized/router.db 
YJYHX_255.22:vrp:10.50.255.22:hradmin:Asdf!123

# 普通配置
[root@LYGVLTOPOT01 ~]# vi /data/oxidized/config
---
username: username
password: password
model: junos
resolve_dns: true
interval: 3600
use_syslog: false
log: /home/oxidized/.config/oxidized/logs/oxidized.log
debug: false
run_once: false
threads: 30
use_max_threads: false
timeout: 20
retries: 3
prompt: !ruby/regexp /^([\w.@-]+[#>]\s?)$/
rest: 0.0.0.0:8888
next_adds_job: false
vars: {}
groups: {}
group_map: {}
models: {}
pid: "/home/oxidized/.config/oxidized/pid"
crash:
  directory: "/home/oxidized/.config/oxidized/crashes"
  hostnames: false
stats:
  history_size: 10
input:
  default: ssh, telnet
  debug: false
  ssh:
    secure: false
  ftp:
    passive: true
  utf8_encoded: true
output:
  default: file
  file:
    directory: "/home/oxidized/.config/oxidized/configs"
source:
  default: csv
  csv:
    file: "/home/oxidized/.config/oxidized/router.db"
    delimiter: !ruby/regexp /:/
    map:
      name: 0
      model: 1
      ip: 2
      username: 3
      password: 4
    gpg: false
model_map:
  juniper: junos
  cisco: ios

```
使用密钥rsa验证 首先生成rsa密钥文件
``` bash
# 生成rsa密钥文件
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa.pem
cp /root/.ssh/id_rsa.pem /data/oxidized/id_rsa.pem

# config中调用
[root@LYGVLTOPOT01 ~]# vi /data/oxidized/config

# 全局
······
next_adds_job: false
vars:
  ssh_keys: "/home/oxidized/.config/oxidized/.ssh/id_rsa.pem"
groups: {}
group_map: {}
······


# 每个节点
map:
  name: 0
  model: 1
  ip: 2
  username: 3
  password: 4
vars_map:
  ssh_keys: 5

YJYHX_255.22:vrp:10.50.255.22:hradmin:Asdf!123:/home/oxidized/.config/oxidized/.ssh/id_rsa.pem
```

## 备份文件自动上传
### 脚本上传
编写脚本，自动上传 安装git
``` bash
yum -y install git

cd /data/oxidized/configs
git init 
git status

git remote add origin https://codehub.hengrui.com/qiuf9/oxidized.git
git add -A
git commit -m "上传"
git push -u origin master 

# 上传会要求输入密码 开启后会自动存储
git config --global credential.helper store


[root@LYGVLK8SW09 ~]# crontab -l
0 * * * * cd /opt;./gitpush.sh > gitpush.log 2>&1

cat /opt/gitpush.sh
#!/bin/bash
source /etc/profile
# FileName:    gitpush.sh 
# Describe:    Used for git push
# Revision:    1.0
# Date:        2024/01/18
# Author:      Austines

dt=`date +%Y%m%d_%H%M`
echo "Backup Begin Date:" $(date +"%Y-%m-%d %H:%M:%S")

cd /data/oxidized/configs
git add -A
git commit -m "Auto commit - $dt"
git push -u origin master
```

### oxidized自动上传
备份文件自动上传至gitlab平台  git产生git基础配置，hooks上传使用
``` bash
output:
  default: git
  file:
    directory: "/home/oxidized/.config/oxidized/configs"
  git:
    single_repo: true
    user: qiuf9
    email: fuqi.qiu.fq9@hengrui.com
    repo: /home/oxidized/.config/oxidized/configs/configs.git
hooks:
  push_to_remote:
    type: githubrepo
    events: [post_store]
    remote_repo: "https://qiuf9:glpat-bxBEWW7gWnUKxYFcB9KA@codehub.hengrui.com/qiuf9/oxidized.git"
```
## 启动oxidized
重启对应的docker镜像名称
``` bash
docker restart oxidized
```
![](https://qiufuqi.github.io/img/hexo/20240813085746.png)

# 问题处理
## 页面展示问题
页面默认不展示某个字段，每次需要点击按钮选中，很麻烦
``` bash
# 进入镜像
docker exec -it oxidized /bin/bash

# 进入目录下，修改 nodes.haml 文件 （具体路径可能有差别）
vi /var/lib/gems/3.0.0/gems/oxidized-web-0.13.1/lib/oxidized/web/views/nodes.haml

# 在底部找到, 删除targets 即可
columnDefs: [{ visible: false, targets: 1}]

```
## 时间问题
docker oxidized时区问题，时间显示不是北京时间 问题原因：ruby语言的时间直接获取的UTC时间，修改完重启镜像
### 更改Last Update时间
- docker exec -it oxidized /bin/bash
- vi /var/lib/gems/3.0.0/gems/oxidized-0.30.1/lib/oxidized/job.rb
- 执行:%s/Time.now.utc/Time.now，把Time.now.utc全部改成Time.now，一共3处
- 退出容器
- 重启容器

![](https://qiufuqi.github.io/img/hexo/20240813113453.png)

### 更改Last Changed时间
- docker exec -it oxidized /bin/bash
- vi /var/lib/gems/3.0.0/gems/oxidized-0.30.1/lib/oxidized/node/stats.rb
- 45行左右，把Time.now.utc全部改成Time.now，一共3处
- 退出容器
- 重启容器

![](https://qiufuqi.github.io/img/hexo/20240814141736.png)


