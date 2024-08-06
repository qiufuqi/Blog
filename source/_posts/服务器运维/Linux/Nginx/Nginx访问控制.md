---
title: Nginx访问控制
date: 2023-12-08
tags:
  - Linux
  - Nginx
categories: 
- 运维
- Nginx
keywords: 'Linux,Nginx'
description: Nginx访问控制
cover: https://qiufuqi.github.io/img/hexo/20221026153151.png
abbrlink: nginx_control
comments: false
---

需求：部分域名要求指定时间不对外访问，指定要配置Nginx

只需在需要限制的server里添加如下配置，重载即可
```bash
# 获取本地时间
if ( $time_local ~ "^(\d+)\/(\w+)\/(\d+):(\d+):(\d+):(\d+) \+(\d+)" ) {
  set $hour $4;
}
# 指定时间黑名单，如果为指定时间，返回500
if ( $hour ~ 00|01|05|06|14 ) {
  return 500;
}
# 重载nginx
nginx -s reload
```


