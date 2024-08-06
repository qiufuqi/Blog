---
title: Linux更新升级curl版本
date: 2023-12-14
tags:
  - Linux
  - CentOS
  - curl
categories: 
- 运维
- curl
keywords: 'Linux,CentOS,curl'
cover: https://qiufuqi.github.io/img/hexo/20231214171849.png
abbrlink: centos_curl
comments: false
---

# 前景
由于低版本的curl存在一定的漏洞，会对我们的服务器安全造成问题，所以，我们需要将curl由低版本安装到高版本。

# 步骤
## 查看当前curl版本
检测服务器安装的curl版本 curl --version
```bash
[root@localhost ~]# curl --version
curl 7.29.0 (x86_64-redhat-linux-gnu) libcurl/7.29.0 NSS/3.53.1 zlib/1.2.7 libidn/1.28 libssh2/1.8.0
Protocols: dict file ftp ftps gopher http https imap imaps ldap ldaps pop3 pop3s rtsp scp sftp smtp smtps telnet tftp 
Features: AsynchDNS GSS-Negotiate IDN IPv6 Largefile NTLM NTLM_WB SSL libz unix-sockets 
```
## 查看curl安装包
查看服务器安装的curl的安装包 rpm -qa curl
```bash
[root@localhost ~]# rpm -qa curl
curl-7.29.0-59.el7_9.1.x86_64
```
## 卸载旧版本curl
直接使用yum remove curl卸载，会报错，别的软件依赖，不能卸载，所以必须强制卸载rpm -e --nodeps
```bash
rpm -e --nodeps curl-7.29.0-59.el7_9.1.x86_64
```
## 下载curl包
网站上找[最新的版本](http://curl.haxx.se/download/)，我们下载最新版本7.87.0
```bash
wget https://curl.haxx.se/download/curl-8.5.0.tar.gz
```
![](https://qiufuqi.github.io/img/hexo/20231214171102.png)

## 解压并安装
```bash
tar -zxvf curl-8.5.0.tar.gz
cd curl-8.5.0
./configure --prefix=/usr/local/curl --with-ssl
make
make install
```
## 添加环境变量
```bash
vi /etc/profile

#在文件最后添加以下内容
export PATH=$PATH:/usr/local/curl/bin

source /etc/profile
```
## 查看版本
```bash
[root@localhost curl-8.5.0]# curl --version
curl 8.5.0 (x86_64-pc-linux-gnu) libcurl/8.5.0 OpenSSL/1.0.2k-fips zlib/1.2.7
Release-Date: 2023-12-06
Protocols: dict file ftp ftps gopher gophers http https imap imaps mqtt pop3 pop3s rtsp smb smbs smtp smtps telnet tftp
Features: alt-svc AsynchDNS HSTS HTTPS-proxy IPv6 Largefile libz NTLM SSL threadsafe UnixSockets
[root@localhost curl-8.5.0]# 

```
