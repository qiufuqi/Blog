---
title: Nginx反向代理
date: 2022-10-26
tags:
  - Linux
  - Nginx
categories: 
- 运维
- Nginx
keywords: 'Linux,Nginx'
description: Nginx反向代理
cover: https://qiufuqi.github.io/img/hexo/20221026153151.png
abbrlink: nginx_daili
comments: false
---

Nginx反向代理
[Nginx 1.20.2 安装](/nginx_install)

# 代理Mysql
**stream和http是同级别的，不要放入http里面；不支持不同域名转发不同Mysql的功能。**
nginx.conf添加如下配置:外网13306端口映射到内网IP:3306端口
``` bash
# 多个映射 只需要变更端口
stream {
    upstream mysql_3306 {
        hash $remote_addr consistent;
        server IP:3306 weight=5 max_fails=3 fail_timeout=30s;
    }
    server {
        listen 13306; # 相当于用nginx所在服务器的13306端口，代替了 upstream mysql_up 那里配置的ip与3306端口，当然也可以是同一台机器中的内部转发
        proxy_connect_timeout 10s;
        proxy_timeout 30000s; #设置客户端和代理服务之间的超时时间，如果5分钟内没操作将自动断开。
        proxy_pass mysql_3306;
    }

    # 多个端口映射时
    upstream mysql_3307 {
        hash $remote_addr consistent;
        server IP:3306 weight=5 max_fails=3 fail_timeout=30s;
    }
    server {
        listen 13307; # 相当于用nginx所在服务器的13306端口，代替了 upstream mysql_up 那里配置的ip与3306端口，当然也可以是同一台机器中的内部转发
        proxy_connect_timeout 10s;
        proxy_timeout 30000s; #设置客户端和代理服务之间的超时时间，如果5分钟内没操作将自动断开。
        proxy_pass mysql_3307;
    }
    # 或者
    server {
        listen 10036;
        proxy_connect_timeout 5s;
        proxy_timeout 300s;
        proxy_pass 10.11.7.87:3306 ;
    }

}
```
# 代理NTP
ntp时间同步服务使用端口123，协议UDP
``` bash
# 多个映射 只需要变更端口
stream {
    upstream ntp_123 {
        server 10.11.7.22:123;
    }
    server {
        listen 123 udp; #由于ntp使用udp协议，这里采用udp协议
        proxy_pass ntp_123;
    }
}
```