---
title: Linux字符处理
date: 2023-5-30
tags:
  - Linux
  - CentOS
  - Shell
  - 字符处理
categories: 
- 运维
- Linux
- 字符处理
keywords: 'Linux,CentOS,字符处理'
cover: https://qiufuqi.github.io/img/hexo/20230418114913.png
abbrlink: centos_char
comments: false
---

# Linux字符处理
93页
## 管道符
| 把上一个命令的输出内容当作下一个命令的输入内容，两个命令之间通过管道符连接即可。
``` bash
# 将输出的内容作为下一个命令more的输入
[root@localhost hard]# ls -l /etc/init.d | more
```
## grep搜索文本
grep文本搜索工具，显示符合条件的所在行。用法如下：
grep [-ivnc] '需要匹配的字符' 文件名
-i ：不区分大小写
-c ：统计包含匹配的行数
-n ：输出行号
-v ：反向匹配
``` bash
[root@localhost home]# cat cat.txt 
The cat's name is Tom, what's the mouse's name?
The mouse's NAME is Jerry
They are good friends
[root@localhost home]# grep -i 'name' cat.txt     # 不区分大小写
The cat's name is Tom, what's the mouse's name?
The mouse's NAME is Jerry
[root@localhost home]# grep -vi 'name' cat.txt    # 不区分大小写和反向匹配
They are good friends
[root@localhost home]# cat cat.txt |grep -vi 'name' # 或者使用管道符
They are good friends
```
## sort排序
sort对无序的数据进行排序，用法如下：
sort [-ntkr] 文件名
-n : 采用数字排序
-t : 指定分隔符
-k : 指定第几列
-r : 反向排序
``` bash
[root@localhost home]# cat cat.txt 
b:3
c:2
a:4
e:5
d:1
f:11
# 指定分隔符：，指定列2，反向排序，数字排序
[root@localhost home]# cat cat.txt | sort -t ":" -k 2 -r -n
f:11
e:5
a:4
b:3
c:2
d:1
```
## uniq删除重复内容
uniq删除重复的行(重复的行需要连续，所以一般和sort连用)，还可以统计出完全相同行出现的总次数,用法如下：
uniq [-ic]
-i : 忽略大小写
-c : 计算重复行数
``` bash
[root@localhost home]# cat cat.txt 
abc
123
abc
456
[root@localhost home]# cat cat.txt | uniq   # 重复行不连续，所以不删除
abc
123
abc
456
[root@localhost home]# cat cat.txt | sort |uniq -c  # 重复行删除，且计算行数
      1 123
      1 456
      2 abc
```
## cut截取文本
cut的处理对象时一行文本，从中选取需要的部分，用法如下：
cut -f 指定的列 -d '分隔符'
``` bash
# 以：作为分隔符，打印出第1列，第6-7列
[root@localhost home]# cat /etc/passwd| cut -f 1,6-7 -d ':'
root:/root:/bin/bash
bin:/bin:/sbin/nologin
daemon:/sbin:/sbin/nologin
adm:/var/adm:/sbin/nologin
lp:/var/spool/lpd:/sbin/nologin
```
cut -c 指定列的字符
``` bash
# 打印出每列1-5和7-10个字符的内容
[root@localhost home]# cat /etc/passwd| cut -c1-5,7-10
root::0:0
bin:x1:1:
daemo:x:2
adm:x3:4:
lp:x::7:l
sync::5:0
```
## tr文本转换
tr命令用于文本转换或删除，用法如下：
tr '转换前' '转换后'
``` bash
# 小写转大写
[root@localhost home]# cat /etc/passwd | tr '[a-z]' '[A-Z]'
ROOT:X:0:0:ROOT:/ROOT:/BIN/BASH
BIN:X:1:1:BIN:/BIN:/SBIN/NOLOGIN
DAEMON:X:2:2:DAEMON:/SBIN:/SBIN/NOLOGIN
```
tr -d '被删除内容'
``` bash
[root@localhost home]# cat /etc/passwd | tr -d ':'
rootx00root/root/bin/bash
binx11bin/bin/sbin/nologin
daemonx22daemon/sbin/sbin/nologin
```
## paste文本合并
paste将文件按照行进行合并，中间默认使用tab隔开，可用-d指定分隔符
paste -d 指定字符  文件1 文件2
``` bash
[root@localhost home]# cat a.txt 
1
2
3
[root@localhost home]# cat b.txt 
a
b
c
[root@localhost home]# paste a.txt b.txt 
1	a
2	b
3	c
[root@localhost home]# paste -d: a.txt b.txt 
1:a
2:b
3:c
```
## split分割大文件
split支持按照行数分割和按照大小分割这两种模式。二进制文件没有行的概念，只能按照大小进行分割。
按照行数：split -l 500 文件名 文件名_
按照大小：split -b 50k 文件名 文件名_
``` bash
[root@localhost home]# split -l 1 a.txt a_
[root@localhost home]# ls
a_aa  a_ab  a_ac
```












