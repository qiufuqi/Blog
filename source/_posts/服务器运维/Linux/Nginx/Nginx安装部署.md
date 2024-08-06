---
title: Nginx安装部署
date: 2022-10-26
tags:
  - Linux
  - Nginx
categories: 
- 运维
- Nginx
keywords: 'Linux,Nginx'
description: Nginx安装部署
cover: https://qiufuqi.github.io/img/hexo/20221026153151.png
abbrlink: nginx_install
comments: false
---

Linux环境中安装nginx（1.20.2），系统版本centos7.6
Nginx 1.20.2 安装
安装方式如下：
- YUM安装 -- 推荐使用yum安装
- 编译安装
- [docker安装](/docker_nginx)
- 脚本安装

# YUM安装
yum安装：官方源安装，epol安装 二选一
## 添加YUM源 
官方源安装
链接: http://nginx.org/en/linux_packages.html#RHEL-CentOS 可以对照自己的系统进行添加
``` bash
[root@master-node ~]# vi /etc/yum.repos.d/nginx.repo
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
gpgcheck=1
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
```
epol安装 (不建议,只能默认版本)
``` bash
[root@master-node ~]# wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
```
## 加载yum包
``` bash
[root@master-node ~]# yum clean all
Loaded plugins: fastestmirror
Cleaning repos: base extras nginx-stable updates
Cleaning up list of fastest mirrors
Other repos take up 609 k of disk space (use --verbose for details)
[root@master-node ~]# yum makecache
```
## 安装指定版本
``` bash
[root@master-node ~]# yum list nginx --showduplicates
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.nju.edu.cn
 * extras: mirrors.nju.edu.cn
 * updates: mirrors.nju.edu.cn
Available Packages
nginx.x86_64           1:1.8.0-1.el7.ngx              nginx-stable
·········
nginx.x86_64           1:1.20.1-1.el7.ngx             nginx-stable
nginx.x86_64           1:1.20.2-1.el7.ngx             nginx-stable
nginx.x86_64           1:1.22.0-1.el7.ngx             nginx-stable

[root@master-node ~]# yum -y install nginx-1.20.2-1.el7.ngx
```
## 查看版本
``` bash
[root@master-node ~]# nginx -v
nginx version: nginx/1.20.2
[root@master-node ~]# rpm -qi nginx
Name        : nginx
Epoch       : 1
Version     : 1.20.2
·········
nginx [engine x] is an HTTP and reverse proxy server, as well as
a mail proxy server.
```
## 启动nginx
启动nginx 和 开机自启
``` bash
[root@master-node ~]# systemctl start nginx
[root@master-node ~]# systemctl enable nginx
Created symlink from /etc/systemd/system/multi-user.target.wants/nginx.service to /usr/lib/systemd/system/nginx.service.
```
## 默认配置
``` bash
[root@master-node ~]# rpm -qc nginx
/etc/logrotate.d/nginx
/etc/nginx/conf.d/default.conf
/etc/nginx/fastcgi_params
/etc/nginx/mime.types
/etc/nginx/nginx.conf
/etc/nginx/scgi_params
/etc/nginx/uwsgi_params
```
## 基本操作
``` bash
# 检测nginx配置是否正确
[root@master-node ~]# nginx -t
nginx: the configuration file /etc/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/conf/nginx.conf test is successful

# 查看nginx进程 并实现不中断服务加载配置
[root@master-node ~]# ps -ef|grep nginx
root     12483     1  0 15:52 ?        00:00:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
nginx    12484 12483  0 15:52 ?        00:00:00 nginx: worker process
nginx    12485 12483  0 15:52 ?        00:00:00 nginx: worker process
nginx    12486 12483  0 15:52 ?        00:00:00 nginx: worker process
nginx    12487 12483  0 15:52 ?        00:00:00 nginx: worker process
root     12553 12296  0 15:56 pts/0    00:00:00 grep --color=auto nginx
[root@master-node ~]# kill -HUP 12483   

[root@localhost ~]# rpm -Uvh http://nginx.org/packages/centos/7/x86_64/RPMS/nginx-1.18.0-1.el7.ngx.x86_64.rpm // rpm方式升级并安装某个版本的Nginx
```

# 编译安装
编译安装优势
- 能够实现定制功能，需要什么功能就可以使用参数加上
- 可以指定安装的路径

## 获取安装包
下载[安装包](http://nginx.org/download/)并解压
``` bash
[root@slave-node opt]# wget http://nginx.org/download/nginx-1.20.2.tar.gz
[root@slave-node opt]# tar -zxvf nginx-1.20.2.tar.gz
```
## 安装依赖包
``` bash
[root@slave-node opt]# yum -y install pcre-devel zlib-devel gcc gcc-c++ make
```
## 创建用户、组
``` bash
[root@slave-node opt]# useradd -M -s /sbin/nologin nginx
```
## 编译安装
安装路径/etc/nginx
``` bash
[root@slave-node nginx-1.20.2]# cd /opt/nginx-1.20.2
[root@slave-node nginx-1.20.2]# ./configure --help
--with-http_ssl_module # 配置HTTPS时使用
--with-http_v2_module # 配置GOLANG语言时使用
--with-stream # 启用TCP/UDP代理服务

[root@slave-node nginx-1.20.2]# ./configure --prefix=/etc/nginx --user=nginx --group=nginx --with-http_stub_status_module

./configure \
--prefix=/etc/nginx \				#指定nginx的安装路径
--user=nginx \										#指定用户名
--group=nginx \										#指定组名
--with-http_stub_status_module						#启用 http_stub_status_module 模块以支持状态统计

[root@slave-node nginx-1.20.2]# make && make install

#让系统识别nginx的操作命令
[root@slave-node nginx-1.20.2]# ln -s /etc/nginx/sbin/nginx /usr/local/sbin/
```
## 添加系统服务
``` bash
[root@slave-node nginx]# vi /lib/systemd/system/nginx.service

[Unit]
Description=nginx
After=network.target
[Service]
Type=forking
PIDFile=/etc/nginx/logs/nginx.pid
ExecStart=/etc/nginx/sbin/nginx
ExecrReload=/bin/kill -s HUP $MAINPID
ExecrStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true
[Install]
WantedBy=multi-user.target

#赋权，除了root以外的用户都不能修改
[root@slave-node nginx]# chmod 754 /lib/systemd/system/nginx.service
```
## 启动nginx
启动nginx 和 开机自启
``` bash
[root@master-node ~]# systemctl start nginx
[root@master-node ~]# systemctl enable nginx
Created symlink from /etc/systemd/system/multi-user.target.wants/nginx.service to /usr/lib/systemd/system/nginx.service.
```
## 常见错误
- ./configure: error: the HTTP rewrite module requires the PCRE library.
``` bash
yum install pcre pcre-devel -y	
```
- ./configure: error: SSL modules require the OpenSSL library.
``` bash
yum install openssl openssl-devel -y 
```
# Docker安装
docker安装nginx参考：[安装参考](/docker_nginx)
# 脚本安装
Nginx的全自动编译安装，安装完成后自动启动并设置开机自启。脚本支持CentOS系列发行版本，shell脚本源码如下：安装版本1.23
``` bash
#!/bin/bash
ck_ok()
{
        if [ $? -ne 0 ]
        then
                echo "$1 error."
                exit 1
        fi
}

download_ng()
{
    cd  /usr/local/src
    if [ -f nginx-1.23.0.tar.gz ]
    then
        echo "当前目录已经存在nginx-1.23.0.tar.gz"
        echo "检测md5"
        ng_md5=`md5sum nginx-1.23.0.tar.gz|awk '{print $1}'`
        if [ ${ng_md5} == 'e8768e388f26fb3d56a3c88055345219' ]
        then
            return 0
        else
            sudo /bin/mv nginx-1.23.0.tar.gz nginx-1.23.0.tar.gz.old
        fi
    fi

    sudo curl -O http://nginx.org/download/nginx-1.23.0.tar.gz
    ck_ok "下载Nginx"
}
install_ng()
{
    cd /usr/local/src
    echo "解压Nginx"
    sudo tar zxf nginx-1.23.0.tar.gz
    ck_ok "解压Nginx"
    cd nginx-1.23.0


    echo "安装依赖"
    if which yum >/dev/null 2>&1
    then
        ## RHEL/Rocky
        for pkg in gcc make pcre-devel zlib-devel openssl-devel
        do
            if ! rpm -q $pkg >/dev/null 2>&1
            then
                sudo yum install -y $pkg
                ck_ok "yum 安装$pkg"
            else
                echo "$pkg已经安装"
            fi
        done
    fi


    if which apt >/dev/null 2>&1
    then
        ##ubuntu
        for pkg in make libpcre++-dev  libssl-dev  zlib1g-dev
        do
            if ! dpkg -l $pkg >/dev/null 2>&1
            then
                sudo apt install -y $pkg
                ck_ok "apt 安装$pkg"
            else
                echo "$pkg已经安装"
            fi
        done
    fi

    echo "configure Nginx"
    sudo ./configure --prefix=/usr/local/nginx  --with-http_ssl_module
    ck_ok "Configure Nginx"


    echo "编译和安装"
    sudo make && sudo make install
    ck_ok "编译和安装"


    echo "编辑systemd服务管理脚本"


    cat > /tmp/nginx.service <<EOF
[Unit]
Description=nginx - high performance web server
Documentation=http://nginx.org/en/docs/
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/bin/sh -c "/bin/kill -s HUP \$(/bin/cat /usr/local/nginx/logs/nginx.pid)"
ExecStop=/bin/sh -c "/bin/kill -s TERM \$(/bin/cat /usr/local/nginx/logs/nginx.pid)"

[Install]
WantedBy=multi-user.target
EOF

    sudo /bin/mv /tmp/nginx.service /lib/systemd/system/nginx.service
    ck_ok "编辑nginx.service"

    echo "加载服务"
    sudo systemctl unmask nginx.service
    sudo  systemctl daemon-reload
    sudo systemctl enable nginx
    echo "启动Nginx"
    sudo systemctl start nginx
    ck_ok "启动Nginx"
}

download_ng
install_ng
```
