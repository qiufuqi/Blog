---
title: CentOS重置开机密码
date: 2022-12-12
tags:
  - Linux
  - CentOS
  - EulerOS
  - 重置密码
categories: 
- 运维
- CentOS
keywords: 'Linux,CentOS,重置密码,EulerOS'
cover: https://qiufuqi.github.io/img/hexo/20221212142328.png
abbrlink: centos_passwd
comments: false
---

# CentOS重置密码
**CentOS7系统重置密码和解封账号**
## 重置密码
启动Linux Centos7系统，当出现如下画面时，直接按“e”键继续。
![](https://qiufuqi.github.io/img/hexo/20221212142556.png)
按向下箭头，一直下滑直至看到如下界面
![](https://qiufuqi.github.io/img/hexo/20221212142610.png)
在如下截图的位置，添加“rw single init=/bin/bash”，添加后按“Ctrl + x”引导系统。
![](https://qiufuqi.github.io/img/hexo/20221212142624.png)
在如下截图位置，即可输入“自定义新密码”来重置root密码了。
![](https://qiufuqi.github.io/img/hexo/20221212142640.png)
运行命令“exec /sbin/init”来正常启动系统，需要输入修改后的root密码。
![](https://qiufuqi.github.io/img/hexo/20221212142655.png)
进入系统后，输入命令“reboot”即可重启系统，重启之后输入用户名和修改以后的密码即可正常进入了。

## 账户锁定
如果因为输错密码导致账户被锁定，需要解封账号

将虚拟机开机，马上按上键或者下键，选中内核按下e，进入下一界面。
![](https://qiufuqi.github.io/img/hexo/20221212142900.png)
将linux 16后面的console内容删除（不删除可能卡住），并在这段最后加上rd.break。若想看开机的过程，可将rhgb quiet删除。
![](https://qiufuqi.github.io/img/hexo/20221212142912.png)
![](https://qiufuqi.github.io/img/hexo/20221212142921.png)
修改完毕后，执行ctrl-x来进入紧急模式，成功后如下图。
![](https://qiufuqi.github.io/img/hexo/20221212142938.png)
依次执行命令
``` bash
mount -o remount,rw /sysroot
chroot /sysroot

pam_tally2 --user=root
pam_tally2 --user=root --reset
pam_tally2 --user=root

passwd         # 输入密码
exit
reboot
```

# EulerOS重置密码
华为欧拉系统EulerOS忘记root密码解决办法

先重启Euler系统，到内核选择界面，按“e”进入紧急模式。
![](https://qiufuqi.github.io/img/hexo/20230413133822.png)
输入救援模式的账号root和密码，默认为**root/Huawei#12**
![](https://qiufuqi.github.io/img/hexo/20230413133900.png)
找到linux开头的的地方，在最后面添加systemd.debug-shell=1，然后按ctrl+x进入内存文件系统。
![](https://qiufuqi.github.io/img/hexo/20230413133925.png)
按完ctrl+x后系统会自动运行，进到正常登录界面，别慌，然后按Ctrl+Alt+F9即可直接免密登录系统，然后修改root密码，重启即可。
![](https://qiufuqi.github.io/img/hexo/20230413133937.png)
