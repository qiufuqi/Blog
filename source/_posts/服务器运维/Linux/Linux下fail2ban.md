---
title: Linux下fail2ban
date: 2023-5-25
tags:
  - Linux
  - CentOS
  - fail2ban
categories: 
- 运维
- fail2ban
keywords: 'Linux,CentOS,fail2ban'
cover: https://qiufuqi.github.io/img/hexo/20230418114913.png
abbrlink: centos_fail2ban
comments: false
---

# fail2ban简介
Fail2Ban 是一款入侵防御软件，可以保护服务器免受暴力攻击。 它是用 Python 编程语言编写的。
Fail2Ban 基于auth 日志文件工作，默认情况下它会扫描所有 auth 日志文件，如 /var/log/auth.log、/var/log/apache/access.log 等，并禁止带有恶意标志的IP，比如密码失败太多，寻找漏洞等等标志。
通常，Fail2Ban 用于更新防火墙规则，用于在指定的时间内拒绝 IP 地址。 它也会发送邮件通知。
Fail2Ban 为各种服务提供了许多过滤器，如 ssh、apache、nginx、squid、named、mysql、nagios 等。
Fail2Ban 能够降低错误认证尝试的速度，但是它不能消除弱认证带来的风险。

# 安装fail2ban
``` bash
[root@localhost ~]# yum -y install epel-release
[root@localhost ~]# yum -y update
[root@localhost ~]# yum -y install fail2ban-firewalld

# 业务参考
 cat /etc/fail2ban/jail.conf				               ## 根据指引，修改配置应在 jail.d/ 下新建文件进行
 vim /etc/fail2ban/jail.d/sshd.local                 ## 修改配置
 cd /etc/fail2ban/jail.d
 vim /etc/fail2ban/action.d/mail-whois.conf	         ## 定义 action
 fail2ban-client reload                              ## 让配置生效 
```
/etc/fail2ban/filter.d/ --- 监狱规则存放处
/etc/fail2ban/jail.d/ --- 启用规则

## 启用邮件提醒功能
安装部署对应邮件发送软件---如不需要可不做
``` bash
[root@localhost ~]# yum -y install sendmail mailx
[root@localhost ~]# systemctl status sendmail # 查看sendmail运行状态
[root@localhost ~]# systemctl start sendmail # 启动
[root@localhost ~]# systemctl enable sendmail # 设置开机自启
[root@localhost ~]# systemctl is-enabled sendmail # 查看是否设置开机自启
```
邮件提醒配置，126和163邮箱都可以
``` bash
[root@localhost ~]# cat /etc/mail.rc
set from=XXXXX@126.com
set smtp=smtps://smtp.126.com:465
set smtp-auth-user=XXXXX@126.com
set smtp-auth-password=XXXXXXXXXXX
set smtp-auth=login

set ssl-verify=ignore                 ##忽略证书警告
set nss-config-dir=/etc/pki/nssdb     ## 证书所在目录
```
发送测试邮件
``` bash
[root@localhost ~]# echo "content" | mail -s "title" xxxxx@qq.com
```

# 配置fail2ban-sshd
安装后，最好将默认的配置文件jail.conf另外拷贝一份。
借助 Fail2Ban 可以筛选出发送这些请求的 IP 地址来进行拦截屏蔽处理

## 编辑sshd规则
最佳做法是使用* .local文件覆盖系统默认值，而不是直接修改jail.config

如果同时存在/etc/fail2ban/jail.local和/etc/fail2ban/jail.d/sshd.local文件，**关于对ssh的约束，后者的优先级是高于前者的**

在最后10分钟内尝试5次后，IP将被阻止10分钟
- ignoreip：永远不会被禁止的IP地址白名单。他们拥有永久的“摆脱监狱”的特权。本地主机的IP地址127.0.0.1是在列表中默认情况下，IPv6的本机地址是::1。如果确认永远不应禁止的其它IP地址，请将它们添加到此列表中，并在每个IP地址之间留一个空格
- bantime：禁止IP地址的时长，m代表分钟、h表示时。如果配置的数值不带m或h，则将其视为秒。值 -1将永久禁止IP地址。要非常小心，不要将自己的计算机给关了起来，这是非常有可能发生的低级错误。
- findtime： 尝试失败的连接次数过多会导致IP地址进“监狱”的时间。
- maxretry: 尝试失败次数过多”的数值。
- backend :指定用于获取文件修改的后端。 如果您使用的是 CentOS 或 Fedora，则需要将后端设置为 systemd。 对于其他操作系统，默认值 auto 就足够了。
  
要设置区别于全局配置文件jail.local的特定配置，必须在/etc/fail2ban/jail.d目录下创建一个相应的“ jail”文件。 对于SSHD，请创建一个名为sshd.local的新文件，然后在其中输入服务过滤说明, 下面就是配置ssh加固的配置文件,该配置的意思是在最后1分钟内尝试5次后，IP将被阻止一年：
``` bash
# 定位到285行左右，添加一行 enabled = true
[root@localhost fail2ban]# cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

# 完成之后修改sshd策略
[root@localhost fail2ban]# vi /etc/fail2ban/jail.d/sshd.local

[sshd]
enabled = true    # 极端情况下，此行即可
port = ssh        # 当我们更改ssh的端口时使用此选项，sshd的默认端口为22，
#action = firewallcmd-ipset
logpath = %(sshd_log)s
maxretry = 5      # 最大尝试次数
bantime = 1800  #封禁时间，单位s。-1为永久封禁
```

## 重启fail2ban
``` bash
[root@localhost ~]# systemctl restart fail2ban
[root@localhost ~]# systemctl status fail2ban
[root@localhost ~]# fail2ban-client status 
Status
|- Number of jail:	1
`- Jail list:	sshd
```
## 测试fail2ban-sshd
尝试错误登录服务器5次，发现再也登录不上了，服务器返回连接超时
``` bash
[root@localhost ~]# fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed:	0
|  |- Total failed:	5
|  `- Journal matches:	_SYSTEMD_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned:	0
   |- Total banned:	1
   `- Banned IP list:	10.12.172.12
```
## 释放被封禁的IP
``` bash
[root@localhost ~]# fail2ban-client set sshd unbanip 0.12.172.12
```


# 配置fail2ban-nginx
## 编辑nginx访问规则
打开编辑 nginx-not-found.conf 监狱规则文件，注意一定要在/etc/fail2ban/filter.d/目录内
``` bash
[root@localhost fail2ban]# vi /etc/fail2ban/filter.d/nginx-not-found.conf
[Definition]
failregex = ^<HOST>.*"(GET|POST).*" (404|444|403|400) .*$
ignoreregex =
```
打开编辑 jail.local 启用这个监狱规则。
``` bash
[root@localhost fail2ban]# vi /etc/fail2ban/jail.d/nginxno404.local
[nginxno404]
#处理 nginx 下的恶意 404 结果扫描
enabled = true
port = http,https
filter = nginx-not-found  # 使用规则 --- filter.d文件夹下
action = iptables[name=nginxno404, port=http, protocol=tcp]
#Fail2Ban 要监控的站点日志文件，大家可以根据自己站点来灵活调整。
logpath  = /usr/local/nginx/logs/access.log
           /usr/local/nginx/logs/error.log
bantime = 3600 #默认是屏蔽 IP 地址 10 分钟
#下面这两个是说 60 秒内 5 次 404 失败请求就开始屏蔽这个 IP 地址
findtime = 60
maxretry = 5
```
**重启fail2ban**
我们换需要验证一下 nginx-not-found.conf 这个监狱规则是否能够生效也就是真正的发现日志文件中返回码是：404、444、403、400 中任意一个的记录，我们可以使用 fail2ban-regex 命令来验证这个规则


``` bash
[root@localhost ~]# systemctl restart fail2ban
[root@localhost ~]# systemctl status fail2ban
[root@localhost ~]# fail2ban-regex /usr/local/nginx/logs/error.log /etc/fail2ban/filter.d/nginx-not-found.conf
Running tests
=============
Use   failregex filter file : nginx-not-found, basedir: /etc/fail2ban
Use         log file : /usr/local/nginx/logs/error.log
Use         encoding : UTF-8
Results
=======
Failregex: 0 total
Ignoreregex: 0 total
Date template hits:
|- [# of hits] date format
|  [32] {^LN-BEG}ExYear(?P<_sep>[-/.])Month(?P=_sep)Day(?:T|  ?)24hour:Minute:Second(?:[.,]Microseconds)?(?:\s*Zone offset)?
`-
Lines: 32 lines, 0 ignored, 0 matched, 32 missed
[processed in 0.01 sec]
Missed line(s): too many to print.  Use --print-all-missed to print all 32 lines
```
## 测试fail2ban-nginx
快速访问不存在的文件，会封禁IP
``` bash
[root@localhost fail2ban]# fail2ban-client status nginxno404
Status for the jail: nginxno404
|- Filter
|  |- Currently failed:	0
|  |- Total failed:	0
|  `- File list:	/usr/local/nginx/logs/error.log /usr/local/nginx/logs/access.log
`- Actions
   |- Currently banned:	0
   |- Total banned:	0
   `- Banned IP list: 10.12.172.12
```

## 释放被封禁的IP
``` bash
[root@localhost ~]# fail2ban-client set nginxno404  unbanip 10.12.172.12

#手动封禁某个ip
[root@localhost ~]# fail2ban-client set nginxno404 banip 10.12.172.12
```


# 忽略某个iP
set <JAIL> addignoreip <IP>	设置某个监控（监狱）可以忽略的ip
set <JAIL> delignoreip <IP>	删除某个监控（监狱）可以忽略的ip