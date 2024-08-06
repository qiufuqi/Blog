---
title: 磁盘inode节点被占满问题
date: 2023-4-17
tags:
  - Linux
  - CentOS
  - inode
categories: 
- 运维
- inode
keywords: 'Linux,CentOS,inode'
cover: https://qiufuqi.github.io/img/hexo/20230417142330.png
abbrlink: centos_inode
comments: false
---
问题：

Linux服务器，查看日志发现程序无法继续写文件，但是用df -h查看磁盘容量还有剩余。
排查思路：怀疑是机器的inode节点被占满，使用df -i查看磁盘inode节点使用情况，果然是inode节点满了。

进行如下步骤进行排查：
1，df -i查看磁盘节点使用情况，查看到inode节点已满。
2，进入到可能的目录，运行for i in ./*; do echo i;findi | wc -l; done统计当前目录使用节点的情况
3，发现/tmp目录下被大量的小文件占满，联系开发发现是程序bug导致不断生成大量小文件，下周改进，在此之前写一个定时清理的脚本删除文件。

总结：解决inode节点满的一般方法就是删除占用inode节点的异常文件