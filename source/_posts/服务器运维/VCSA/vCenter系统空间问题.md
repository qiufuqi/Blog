---
title: vCenter系统空间问题.md
date: 2022-12-16
tags:
  - EXSI
  - vCenter
categories: 
- 运维
- EXSI
- vCenter
keywords: 'Exsi,vCenter,磁盘空间'
cover: https://qiufuqi.github.io/img/hexo/20220922143047.png
abbrlink: exsi_vCenter_kongjian
comments: false
---

**VC出现（log Disk Exhaustion on vc）磁盘空间满解决方法**

[参考地址](https://blog.csdn.net/yzqtcc/article/details/124577270)
有两种方法：
第一找到对应的存放日志文件目录，删除一些日志包（但是删除日志包毕竟不能彻底解决问题）
![](https://qiufuqi.github.io/img/hexo/20221216170348.png)
或者进入管理后台查看 ip:5480 => 监控 => 磁盘
![](https://qiufuqi.github.io/img/hexo/20221216170704.png)

第二就是对VC相应的日志对应的盘进行扩容
注意：只要是改变原VC状态的操作都会存在风险，扩容不管是在Linux里面是VC存在风险。最好是备份一下（加盘后再去克隆是克隆不了的）

可以看下图所示，/storage/log文件已经到了81%。
![](https://qiufuqi.github.io/img/hexo/20221216170424.png)
找到对应的log_vg-log使用的物理卷，并找到是VC上的第几块盘
![](https://qiufuqi.github.io/img/hexo/20221216170436.png)
然后从原来的10G扩容在20G（依据实际需求来扩容大小）然后重启
![](https://qiufuqi.github.io/img/hexo/20221216170454.png)
增加虚拟磁盘空间后，返回 SSH 会话，然后运行以下命令以自动展开所有增加了物理卷的逻辑卷：
/usr/lib/applmgmt/support/scripts/autogrow.sh
![](https://qiufuqi.github.io/img/hexo/20221216170508.png)
![](https://qiufuqi.github.io/img/hexo/20221216170516.png)
最后查看
![](https://qiufuqi.github.io/img/hexo/20221216170530.png)