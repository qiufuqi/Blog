---
title: Linux用户管理
date: 2023-5-26
tags:
  - Linux
  - CentOS
  - Shell
  - 用户管理
categories: 
- 运维
- Linux
- 用户管理
keywords: 'Linux,CentOS,用户管理'
cover: https://qiufuqi.github.io/img/hexo/20230418114913.png
abbrlink: centos_user
comments: false
---

# Linux用户管理
Linux用户按照使用方式分为三种：一是跟用户，二是系统用户，三是普通用户
## 新增删除用户
### 新增用户
useradd用于添加新的用户，通常情况下在该命令后跟上新增的用户名，比如useradd john即可。
后台创建步骤：
- 一般会在/etc/passwd和/etc/shadow末尾追加一条记录，同时分配给该用户一个UID
- 为该用户创建家目录，创建路径在/home目录中
- 复制/etc/skel下所有文件至家目录中
- 创建一个与该用户名一样的用户组，示例中创建john的用户时也会同时创建john的用户组

创建用户其他参数: -u -g -d
``` bash
[root@localhost ~]# useradd -u 555 user1          #系统指定分配一个UID，该UID不能与其他用户冲突
[root@localhost ~]# useradd -g user1 user2        #创建用户user2时 指定用户所属group是user1
[root@localhost ~]# useradd -d /home/mydir user3  # 创建user3时指定家目录/home/mydir
```
### 修改密码
passwd用于修改用户密码，在不设置密码的情况下，/etc/shadow中该用户的记录第二列显示为两个感叹号!!
root用户passwd可以修改密码，普通用户只能修改自己用户密码，输入两次相同的密码。
``` bash
[root@localhost ~]# passwd john 
```
### 修改用户
usermod用于对用户进行操作，本质上就是对/etc/passwd和/etc/shadow文件做一些修改。
-d 指定目录
-m 指定用户
-L 冻结用户
-U 解冻用户
``` bash
# john的家目录修改为john_new
[root@localhost ~]# usermod -d /home/john_new -m john
[root@localhost ~]# cat /etc/passwd|grep john
john:x:1000:1000::/home/john_new:/bin/bash

# 冻结用户 && 解冻用户
[root@localhost ~]# usermod -L john
[root@localhost ~]# usermod -U john
```
### 删除用户
userdel用于删除某个用户，删除用户时并不会删除原来的家目录，如果要删除家目录添加参数-r
``` bash
[root@localhost ~]# userdel john        # 保留家目录
[root@localhost ~]# userdel -r john     # 删除家目录
```

## 新增和删除用户组
### 增加用户组
groupadd用于创建用户组，后接用户组名作为其参数，记录在/etc/group
``` bash
# 创建用户组，用户组GID为1000
[root@localhost ~]# groupadd john
[root@localhost ~]# cat /etc/group|grep john
john:x:1000:
```
### 删除用户组
groupdel用户删除用户组，后接用户组名作为其参数，该组下没有用户才可以删除。
``` bash
[root@localhost ~]# groupdel john
```

## 检查用户信息
### 查看用户
users,who,w 查看当前系统有哪些用户
``` bash
[root@localhost ~]# users
root root

# who 用户名 终端 登录时间
[root@localhost ~]# who               
root     tty1         2023-05-24 16:31
root     pts/0        2023-05-26 10:55 (10.12.172.12)

# w 第一行  当前时间  系统运行时间  已登录用户数量  系统负载
# w 第二行  用户名  终端  来源  登录时间  闲置时间  运行进程消耗CPU时间总量 闲置进程消耗CPU时间总亮 当前运行进程
[root@localhost ~]# w
 17:03:35 up 2 days, 32 min,  2 users,  load average: 0.00, 0.01, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
root     tty1                      三16   23:44m  0.02s  0.02s -bash
root     pts/0    10.12.172.12     10:55    7.00s  0.31s  0.01s w
```
### 调查用户
finger 用户：显示某个用户详细信息，默认显示系统登录用户
``` bash
[root@localhost ~]# finger
Login     Name       Tty      Idle  Login Time   Office     Office Phone   Host
root      root       tty1    23:52  May 24 16:31           
root      root       pts/0          May 26 10:55                           (10.12.172.12)
[root@localhost ~]# finger ntp
Login: ntp            			Name: 
Directory: /etc/ntp                 	Shell: /sbin/nologin
Never logged in.
No mail.
No Plan.
```

## 切换用户 
linux用户分为根用户，普通用户，系统用户 3种，其中根用户和普通用户是可以登录到系统中。
### 切换其他用户
su 用户：切换用户的意思，默认是切换到root用户下（需要输入root密码），exit可回到之前用户
su - 切换到root时还能应用到root的用户环境，到/root目录下
### 其他身份sudo
sudo + 命令：使用root的身份执行命令。系统会检查/etc/sudoers，判断该用户是否有执行sudo的权限，有的话输入自己用户密码即可执行。
编辑/etc/sudoers文件，可使用vi/vim，或者visudo（保存退出时会自动检查语法）
修改后john就可以使用sudo操作命令。
``` bash
[root@localhost ~]# visudo
·········
## Allow root to run any commands anywhere
root    ALL=(ALL)       ALL
john    ALL=(ALL)       ALL             # john用户（第一列） 任何地方登录（第二列） 执行任何人（第三列） 任何命令（第四列）
%john    ALL=(ALL)       ALL            # john用户组（第一列） 任何地方登录（第二列） 执行任何人（第三列） 任何命令（第四列）
%john    ALL=(ALL)       NOPASSWD:ALL   # 不需要输入密码

# 最后一列 ALL非常危险 可指定命令 sudo /sbin/reboot
john    ALL=(ALL)       NOPASSWD:/sbin/reboot, /usr/bin/reboot  # 用户可冲洗或者关闭服务器
·········
```

## 任务管理
某些指定时间执行指定任务
### 单一时刻任务
at 在指定的时间执行任务，默认所有用户都可以使用at来调度自己的任务，禁用某个用户则将用户名添加到/etc/at.deny中

### 周期性任务
cron


57页








* 6 * * * find /root/home/zhang/* ctime 1 -exec rm -rf {} \;

表示每天早上六点删除/root/home/zhang目录下一天前创建的所有文件，不删除文件夹zhang，如果这个文件夹也要删除的话用* 6 * * * find /root/home/zhang ctime 1 -exec rm -rf {} \;

* 6 * * *

第一个*号表示时间中的 分钟  取值范围：0-59

第二个*号表示时间中的 小时  取值范围：0-23

第三个*号表示一个月中的第几天，取值范围：1-31

第四个*号表示一年中的第几个月，取值范围：1-12

第五个*号表示一个星期中的第几天，以星期天开始依次的取值为0～7，0、7都表示星期天

ctime 表示创建时间，1 表示一天前，其实Linux中不存在文件创建时间，只有访问时间(atime)、修改时间(mtime)、状态改动时间(ctime)

可以通过命令 stat + 文件路径  查看时间

也可通过命令 touch -t 201212212359 aa (建立文件aa,时间是2012年12月21日23时59分)修改时间

若是删除目录下的指定文件可以用：find 对应目录 -mtime +天数 -name "文件名" -exec rm -rf {} \;

写好了命令，下面就是启动定时任务了。













## 统计工具
wc 命令用于计算字数。
参数：
-c, --bytes：统计字节数。
-m, --chars：统计字符数。
-w, --words：统计字数。
-l, --lines：统计行数。
-L, --max-line-length：统计最长行的长度。
``` bash
[root@localhost shell]# wc -l init.sh 
23 init.sh
```

## &
加在一个命令的最后，可以把这个命令放到后台执行

## 文本操作
Linux 文本操作的三大神器：grep、sed、awk，各自的最佳应用场景：
- grep：使用正则表达式搜索文本，并把匹配的行打印出来，是强大的文本搜索工具；
- sed：用于编辑匹配到的文本，是一种流编辑器；
- awk：能够对文本进行复杂的格式处理，是一种处理文本的语言[具体介绍](/centos_awk)

基本命令格式：awk '{pattern + action}'  {filenames}
pattern表示在数据中要查找的内容，action表示要执行的一系列命令。
awk 通过指定分隔符，将一行分为多个字段，依次用 $1、$2 ... $n 表示第一个字段、第二个字段... 第n个字段。

