---
title: Linux下awk命令
date: 2023-4-21
tags:
  - Linux
  - CentOS
  - awk
categories: 
- 运维
- 常用命令
keywords: 'Linux,CentOS,awk'
cover: https://qiufuqi.github.io/img/hexo/20230418114913.png
abbrlink: centos_awk
comments: false
---

Linux下awk基本命令

# 文本操作
Linux 文本操作的三大神器：grep、sed、awk，各自的最佳应用场景：
- grep：使用正则表达式搜索文本，并把匹配的行打印出来，是强大的文本搜索工具；
- sed：用于编辑匹配到的文本，是一种流编辑器；
- awk：能够对文本进行复杂的格式处理，是一种处理文本的语言。

## awk使用方法
- 命令行方式
- shell脚本方式
- 将所有的awk命令插入一个单独文件，然后调用：

https://blog.csdn.net/qq_15245487/article/details/100144279

##  awk 基本操作
awk是一个强大的文本分析工具，相对于grep的查找，sed的编辑，awk在其对数据分析并生成报告时，显得尤为强大。简单来说awk就是把文件逐行的读入，以空格为默认分隔符将每行切片，切开的部分再进行各种分析处理。
通常，**awk是以文件的一行为处理单位的。awk每接收文件的一行，然后执行相应的命令，来处理文本。**

基本命令格式：awk '{pattern + action}'  {filenames}
pattern表示在数据中要查找的内容，action表示要执行的一系列命令。
awk 通过指定分隔符，将一行分为多个字段，依次用 $1、$2 ... $n 表示第一个字段、第二个字段... 第n个字段。
**awk的默认分隔符是空格和制表符**
比如有一log文件，若只想获取 vel、acc、steer 的值，则可以通过下面的命令：
``` bash
[root@localhost home]# cat cat.log 
vel: 2.8, acc: 0.5, steer: 1.5
vel: 3.8, acc: 1.5, steer: 2.5
vel: 4.8, acc: 2.5, steer: 3.5
vel: 5.8, acc: 3.5, steer: 4.5
vel: 6.8, acc: 4.5, steer: 5.5
# awk 默认分隔符是空格和制表符 (vel:) (6.8,) (acc:) (4.5,) (steer:) (5.5)
[root@localhost home]# awk '{print $2, $4, $6}' cat.log 
2.8, 0.5, 1.5
3.8, 1.5, 2.5
4.8, 2.5, 3.5
5.8, 3.5, 4.5
6.8, 4.5, 5.5
```
## awk 的分隔符
awk的默认分隔符是空格和制表符，上面的例子中，若希望把逗号去掉，则可以使用 -F 参数来指定分隔符，命令如下：
awk -F ':|,' '{print $2, $4, $6}' log
这里指定冒号(:)和逗号(,)同时作为分隔符。
``` bash
[root@localhost home]# cat cat.log 
vel: 2.8, acc: 0.5, steer: 1.5
vel: 3.8, acc: 1.5, steer: 2.5
vel: 4.8, acc: 2.5, steer: 3.5
vel: 5.8, acc: 3.5, steer: 4.5
vel: 6.8, acc: 4.5, steer: 5.5
# 冒号(:)和逗号(,)同时作为分隔符。 (vel): (6.8), (acc): (4.5), (steer): (5.5)
[root@localhost home]# awk -F ':|,' '{print $2, $4, $6}' cat.log 
 2.8  0.5  1.5
 3.8  1.5  2.5
 4.8  2.5  3.5
 5.8  3.5  4.5
 6.8  4.5  5.5
```
## awk 的内置变量
除了 $1、$2 ... $n，awk 还有一些内置变量，常用的如下：
- $0：表示当前整行，$1表示第一个字段，$2表示第二个字段，$n 表示第n个字段；
- NR：表示当前已读的行数；
- NF：表示当前行被分割的列数，NF表示最后一个字段，NF-1 表示倒数第二个字段；
- FILENAME：表示当前文件的名称

``` bash
[root@localhost home]# cat cat.log 
vel: 2.8, acc: 0.5, steer: 1.5
vel: 3.8, acc: 1.5, steer: 2.5
vel: 4.8, acc: 2.5, steer: 3.5
vel: 5.8, acc: 3.5, steer: 4.5
vel: 6.8, acc: 4.5, steer: 5.5
[root@localhost home]# awk '{print FILENAME, NR, NF, $0}' cat.log 
cat.log 1 6 vel: 2.8, acc: 0.5, steer: 1.5
cat.log 2 6 vel: 3.8, acc: 1.5, steer: 2.5
cat.log 3 6 vel: 4.8, acc: 2.5, steer: 3.5
cat.log 4 6 vel: 5.8, acc: 3.5, steer: 4.5
cat.log 5 6 vel: 6.8, acc: 4.5, steer: 5.5
```
## awk 条件判断
awk 的 pattern 也支持使用条件判断，比如只打印 vel 小于 4.0 的行，命令如下
awk '$2 < 4.0 {print $0}' log
``` bash
[root@localhost home]# cat cat.log 
vel: 2.8, acc: 0.5, steer: 1.5
vel: 3.8, acc: 1.5, steer: 2.5
vel: 4.8, acc: 2.5, steer: 3.5
vel: 5.8, acc: 3.5, steer: 4.5
vel: 6.8, acc: 4.5, steer: 5.5
[root@localhost home]# awk '$2 < 4.0 {print $0}' cat.log 
vel: 2.8, acc: 0.5, steer: 1.5
vel: 3.8, acc: 1.5, steer: 2.5
```
## awk 统计值计算
awk 还可以快速计算出一些统计值，比如最大值，最小值，平均值等。
比如计算 vel 的最大值，acc 的最小值，steer 的平均值，命令如下：
awk -F ':|,' 'BEGIN {max=0} {if($2>max) max=$2} END {print "max vel:", max}' cat.log 
awk -F ':|,' 'BEGIN {min=10} {if($4<min) min=$4} END {print "min acc:", min}' cat.log 
awk -F ':|,' 'BEGIN {sum=0} {sum+=$6} END {print "steer avg:", sum/NR}' cat.log 
命令中的 BEGIN 和 END 都是awk的关键字：
- BEGIN：表示在awk程序开始前执行一次；
- END：表示在awk程序结束后执行一次

``` bash
[root@localhost home]# cat cat.log 
vel: 2.8, acc: 0.5, steer: 1.5
vel: 3.8, acc: 1.5, steer: 2.5
vel: 4.8, acc: 2.5, steer: 3.5
vel: 5.8, acc: 3.5, steer: 4.5
vel: 6.8, acc: 4.5, steer: 5.5
[root@localhost home]# awk -F ':|,' 'BEGIN {max=0} {if($2>max) max=$2} END {print "max vel:", max}' cat.log 
max vel:  6.8
[root@localhost home]# awk -F ':|,' 'BEGIN {min=10} {if($4<min) min=$4} END {print "min acc:", min}' cat.log 
min acc:  0.5
[root@localhost home]# awk -F ':|,' 'BEGIN {sum=0} {sum+=$6} END {print "steer avg:", sum/NR}' cat.log 
steer avg: 3.5
```
## awk的print和printf
awk 同时支持 print 和 printf 两种打印输出的函数。
- print：其参数可以是变量、数值或字符串，字符串必须用双引号，参数用逗号分开；
- printf：其用法与C语言的printf相似，可以格式化输出。

``` bash
[root@localhost home]# cat cat.log 
vel: 2.8, acc: 0.5, steer: 1.5
vel: 3.8, acc: 1.5, steer: 2.5
vel: 4.8, acc: 2.5, steer: 3.5
vel: 5.8, acc: 3.5, steer: 4.5
vel: 6.8, acc: 4.5, steer: 5.5
[root@localhost home]# awk -F ':|,' '{printf("%.2f %.2f %.2f\n", $2, $4, $6)}' cat.log 
2.80 0.50 1.50
3.80 1.50 2.50
4.80 2.50 3.50
5.80 3.50 4.50
6.80 4.50 5.50
```