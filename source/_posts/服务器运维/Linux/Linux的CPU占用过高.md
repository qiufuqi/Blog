---
title: Linux的CPU占用过高
date: 2023-4-18
tags:
  - Linux
  - CentOS
  - CPU
categories: 
- 运维
- 常用命令
keywords: 'Linux,CentOS,Command'
cover: https://qiufuqi.github.io/img/hexo/20230418114913.png
abbrlink: centos_cpu
comments: false
---

某台服务器上的一个应用总是隔一段时间就自己挂掉 用top看了看 从重新部署应用开始没有多长时间CPU占用上升得很快
参考：https://www.cnblogs.com/you-men/p/13382659.html
# 排查步骤
``` bash
# 1.使用top 定位到占用CPU高的进程PID
top

# 2.通过ps aux | grep PID命令
获取线程信息，并找到占用CPU高的线程
ps -mp pid -o THREAD,tid,time | sort -rn

# 3.将需要的线程ID转换为16进制格式
printf "%x\n" tid

# 4.打印线程的堆栈信息 到了这一步具体看堆栈的日志来定位问题了
jstack pid |grep tid -A 30
```
# 案例
## TOP查看进程
top可以看到PID733进程的占用172%
![](https://qiufuqi.github.io/img/hexo/20230418150959.png)

## 获取线程信息
查找进程733下的线程 可以看到TID 线程775占用了96%且持有了很长时间 其实到这一步基本上能猜测到应该是 肯定是那段代码发生了死循环
ps -mp 733 -o THREAD,tid,time | sort -rn
![](https://qiufuqi.github.io/img/hexo/20230418151127.png)

## 线程ID转换
线程ID转换为16进制格式
printf "%x\n" 775
![](https://qiufuqi.github.io/img/hexo/20230418151203.png)

## 查看java的堆栈信息
jstack 733 |grep 307 -A 30
![](https://qiufuqi.github.io/img/hexo/20230418151347.png)