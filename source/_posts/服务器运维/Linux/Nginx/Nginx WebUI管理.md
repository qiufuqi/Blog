---
title: Nginx WebUI管理
date: 2024-1-11
tags:
  - Linux
  - Nginx
  - WebUI
categories: 
- 运维
- Nginx
- WebUI
keywords: 'Linux,Nginx,WebUI'
description: Nginx安装部署
cover: https://qiufuqi.github.io/img/hexo/20240111162803.png
abbrlink: nginx_WebUI
comments: false
---

Nginx WebUI是一款方便实用的nginx 网页配置工具，可以使用 WebUI 配置 Nginx 的各项功能，包括端口转发，反向代理，ssl 证书配置，负载均衡等，最终生成「nginx.conf」配置文件并覆盖目标配置文件，完成 nginx 的功能配置。**数据库使用h2, 因此服务器上不需要安装任何数据库**
[WebUI官网](https://www.nginxwebui.cn/)
[安装参考](https://www.nginxwebui.cn/product.html)

![](https://qiufuqi.github.io/img/hexo/20240111161641.png)

浏览器访问地址：http://YouIP:8080/adminPage/login
## 安装nginx环境
[nginx安装](/nginx_install)

# jar部署WebUI
## 安装java环境
```bash
[root@localhost ~]# yum search java|grep jdk
[root@localhost ~]# yum -y install java-1.8.0-openjdk 
[root@localhost ~]# java -version
```
## 下载jar包
```bash
[root@localhost ~]# mkdir /home/nginxWebUI/
[root@localhost ~]# wget -O /home/nginxWebUI/nginxWebUI.jar http://file.nginxwebui.cn/nginxWebUI-3.8.2.jar
```
## 启动程序
```bash
nohup java -jar -Dfile.encoding=UTF-8 /home/nginxWebUI/nginxWebUI.jar --server.port=8080 --project.home=/home/nginxWebUI/ > /dev/null &

参数说明(都是非必填)
--server.port 占用端口, 默认以8080端口启动
--project.home 项目配置文件目录，存放数据库文件，证书文件，日志等, 默认为/home/nginxWebUI/
--spring.database.type=mysql 使用其他数据库，不填为使用本地h2，可选mysql
--spring.datasource.url=jdbc:mysql://ip:port/nginxwebui 数据库url
--spring.datasource.username=root 数据库用户
--spring.datasource.password=pass 数据库密码
注意命令最后加一个&号, 表示项目后台运行
```
## 设置开机自启动
```bash
# 配置nginx_webui开机自启
vi /usr/lib/systemd/system/nginx_webui.service
# 写入以下信息：
[Unit]
Description=nginxWebUI
After=network.target

[Service]
Restart=on-failure
ExecStart=/usr/bin/java -jar -Dfile.encoding=UTF-8 /home/nginxWebUI/nginxWebUI.jar --server.port=8080 --project.home=/home/nginxWebUI/ > /dev/null
Restart=on-failure

[Install]
WantedBy=multi-user.target

#重启服务
systemctl daemon-reload
systemctl start nginx_webui
systemctl status nginx_webui
systemctl enable nginx_webui
```


# docker安装
## 安装docker环境
docker环境安装(/docker_install)
## 下载镜像
```bash
[root@localhost ~]# docker pull cym1102/nginxwebui:latest
```
## 启动容器:
```bash
docker run -itd  --restart="always" -v /home/nginxWebUI:/home/nginxWebUI -e BOOT_OPTIONS="--server.port=8080" --privileged=true --net=host cym1102/nginxwebui:latest
```
启动容器时请使用--net=host参数, 直接映射本机端口, 因为内部nginx可能使用任意一个端口, 所以必须映射本机所有端口.
容器需要映射路径/home/nginxWebUI:/home/nginxWebUI, 此路径下存放项目所有数据文件, 包括数据库, nginx配置文件, 日志, 证书等, 升级镜像时, 此目录可保证项目数据不丢失. 请注意备份。
-e BOOT_OPTIONS 参数可填充java启动参数, 可以靠此项参数修改端口号, "--server.port 占用端口", 不填默认以8080端口启动
日志默认存放在/home/nginxWebUI/log/nginxWebUI.log

# 找回密码
如果忘记了登录密码或没有保存两步验证二维码，可按如下教程重置密码和关闭两步验证.
```bash
# 1. 停止nginxWebUI进程或停止docker容器运行
# 2. 使用找回密码参数运行nginxWebUI.jar, docker用户需单独下载nginxWebUI.jar运行此命令
java -jar nginxWebUI.jar --project.home=/home/nginxWebUI/ --project.findPass=true
```
--project.home 为项目文件所在目录, 使用docker容器时为映射目录
--project.findPass 为是否打印用户名密码

运行成功后即可重置并打印出全部用户名密码并关闭两步验证

