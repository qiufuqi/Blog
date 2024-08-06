---
title: CentOS升级Python3
date: 2022-11-17
tags:
  - Linux
  - CentOS
  - Python
categories: 
- 运维
- Python
keywords: 'Linux,CentOS,升级'
cover: https://qiufuqi.github.io/img/hexo/20221117161321.png
abbrlink: python_upgrate
comments: false
---

在linux中有自带的python，但是python的版本是2.7，但是我们需要用到的是python3.x，所以就不得不升级python2.7为python3.x
共计2种方法，推荐使用第二种

## 方法一

### 查看python的版本
``` bash
 python --version  或者  # python -V
```
### 下载新版本
下载最新版本python并解压缩
[官网下载版本](https://www.python.org/downloads/source/)
``` bash
wget https://www.python.org/ftp/python/3.7.7/Python-3.7.7.tgz
tar -zxvf Python-3.7.7.tgz
cd Python-3.7.7/
```
### 开始安装
**如果没有升级过Python就需要安装Python相关的依赖包**
``` bash
yum update -y
yum install -y make gcc gcc-c++
```
安装完依赖包后开始安装python3
``` bash
./configure
make && make install
```
### 版本更换
在/usr/local/bin/下有一个python3的链接，指向bin目录下的python3.7
``` bash
python --version
Python 2.7.16
```
查看python的路径，在/usr/bin下面，可以看到python的链接是python2.7，所以执行python就相当于是执行python2.6
设置3.x为默认版本：
``` bash
 ls -la /usr/bin | grep python
 mv /usr/bin/python /usr/bin/python.bak
```
进入解压的目录当中，有一个python的可执行文件或者使用命令查询
``` bash
find / -name python
/root/Python-3.3.0/python

# 若是输出为3.x，就说明是python3
ln -s /home/ec2-user/Python-3.7.7/python /usr/bin/python
python --version
```
### 配置yum
升级Python版本之后将由默认的python指向了python3，yum不能正常使用，需要更改yum的配置文件
``` bash
# vi /usr/bin/yum
# vi /usr/libexec/urlgrabber-ext-down
```
修改文件内容如下：
``` bash
#!/usr/bin/python ==> #!/usr/bin/python2.7
```
安装pip
下载pip，[官网地址](https://pypi.org/project/pip/)
``` bash
[root@localhost ~]# pip3 install --upgrade pip
```

## 方法二
### yum安装
``` bash
[root@slave-node ~]# yum -y install python3
[root@slave-node ~]# cd /usr/bin/
[root@slave-node bin]# ll|grep python
lrwxrwxrwx. 1 root root         9 Oct 19 15:32 python2 -> python2.7
-rwxr-xr-x. 1 root root      7216 Oct 31  2018 python2.7
lrwxrwxrwx. 1 root root         7 Oct 19 15:32 python2.bak -> python2
lrwxrwxrwx  1 root root         9 Nov 17 16:04 python3 -> python3.6
-rwxr-xr-x  2 root root     11328 Nov 17  2020 python3.6
-rwxr-xr-x  2 root root     11328 Nov 17  2020 python3.6m
[root@slave-node bin]# mv python python2.bak
[root@slave-node bin]# ln -sf python3.6 python

# 只用执行一次，防止yum不可用
[root@slave-node bin]# sed -i 's?#!/usr/bin/python?&2?' /usr/bin/yum
```

## pip:命令报错 解决方法
1、下载
``` bash
# 默认版本，或者指定版本
wget https://bootstrap.pypa.io/get-pip.py

wget https://bootstrap.pypa.io/pip/3.6/get-pip.py
wget https://bootstrap.pypa.io/pip/2.7/get-pip.py
```
2、安装
``` bash
python get-pip.py
```

3、查看pip版本（如果本步骤正常，忽略4/5步）
``` bash
pip -V
```

4、查找pip安装路径
``` bash
find / -name pip
```

5、将pip添加到系统命令
``` bash
ln -s  /usr/local/python/bin/pip /usr/bin/pip
```












