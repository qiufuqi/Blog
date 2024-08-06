---
title: CentOS安装FTP
date: 2022-11-17
tags:
  - Linux
  - CentOS
  - FTP
categories: 
- 运维
- FTP
keywords: 'Linux,CentOS,FTP'
cover: https://qiufuqi.github.io/img/hexo/20230224155332.png
abbrlink: centos_ftp
comments: false
---

**CentOS 7 部署FTP**
[配置参考](https://www.cnblogs.com/staryea/p/8520817.html)
## 安装FTP服务
部署ftp服务，并设置开机自启动
``` bash
[root@localhost ~]# yum install -y vsftpd
[root@localhost ~]# systemctl start vsftpd
[root@localhost ~]# systemctl enable vsftpd
```
## 修改配置文件
``` bash
[root@localhost ~]# cd /etc/vsftpd && cp vsftpd.conf vsftpd.conf.bak
[root@localhost vsftpd]# vi vsftpd.conf
·········
在101行,102,104行
101 chroot_local_user=YES  --改为YES chroot_local_user=YES将所有用户限定在主目录内
102 chroot_list_enable=YES  --改为YES  chroot_list_enable=YES表示要启用chroot_list_file
103 # (default follows)
104 chroot_list_file=/etc/vsftpd/chroot_list --注释放开 chroot_list_file这时列出的是那些“不会被限制在主目录下”的用户。

# 最后追加
userlist_deny=NO  --新增
userlist_enable=YES --默认是YES
```
## 增加用户
root用户执行 创建用户 && 设置密码
``` bash
[root@localhost home]# useradd -d /home/vcenter -g ftp -s /sbin/nologin ftp_vcenter
[root@localhost home]# passwd ftp_vcenter 

# /home/vcenter是ftp_vcenter用户的主目录 
# ftp_vcenter是ftp用户
# -g ftp是ftp组
# 如果创建错误 groupdel 或者userdel
# userdel -r ftp_vcenter
```
## 配置chroot_list文件
ftp_vcenter 代表 这个用户不被限制主目录内
``` bash
[root@localhost home]# vi /etc/vsftpd/chroot_list
ftp_vcenter
```
## 配置允许访问用户
``` bash
[root@localhost home]# vi /etc/vsftpd/user_list
·········
ftp_vcenter
```
## 更改pam.d
注释两个auth
``` bash
[root@localhost pam.d]# cat /etc/pam.d/vsftpd 
#%PAM-1.0
session    optional     pam_keyinit.so    force revoke
#auth       required	pam_listfile.so item=user sense=deny file=/etc/vsftpd/ftpusers onerr=succeed
#auth       required	pam_shells.so
auth       include	password-auth
account    include	password-auth
session    required     pam_loginuid.so
session    include	password-auth
```
## 重启FTP服务
``` bash
[root@localhost home]# service vsftpd restart
```
![](https://qiufuqi.github.io/img/hexo/20230224164210.png)

## 其他
vsftpd.ftpusers：位于/etc/vsftpd目录下。它指定了哪些用户账户不能访问FTP服务器，例如root等。 如果想要root 登录 则注释里面的root
![](https://qiufuqi.github.io/img/hexo/20230224163231.png)

vsftpd.user_list：位于/etc/vsftpd目录下。该文件里的用户账户在默认情况下也不能访问FTP服务器，
仅当vsftpd .conf配置文件里启用userlist_enable=NO选项时才允许访问。默认是YES，代表这个配置文件生效
我们在这里 如果只想让这里面的用户登录到FTP 需要添加 userlist_deny=NO 参数这个参数=NO 代表 这个配置信息的用户可以访问FTP
![](https://qiufuqi.github.io/img/hexo/20230224163327.png)










