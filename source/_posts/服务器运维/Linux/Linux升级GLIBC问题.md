---
title: Linux升级GLIBC问题.md
date: 2024-1-12
tags:
  - Linux
  - CentOS
  - GLIBC
categories: 
- 运维
- GLIBC
keywords: 'Linux,CentOS,GLIBC'
cover: https://qiufuqi.github.io/img/hexo/20230417142330.png
abbrlink: centos_GLIBC
comments: false
---
[GLIBC升级参考](https://www.cnblogs.com/FengZeng666/p/15989106.html)
## 升级GLIBC
当编译时遇到报错，安装升级GLIBC
```bash
./nginx-vtx-exporter: /lib64/libc.so.6: version `GLIBC_2.32' not found (required by ./nginx-vtx-exporter)
./nginx-vtx-exporter: /lib64/libc.so.6: version `GLIBC_2.34' not found (required by ./nginx-vtx-exporter)
```
升级GLIBC到2.34版本
```bash
wget  https://mirror.bjtu.edu.cn/gnu/libc/glibc-2.34.tar.xz
wget http://ftp.gnu.org/gnu/glibc/glibc-2.32.tar.gz

tar -xf glibc-2.34.tar.xz -C /usr/local/
cd /usr/local/glibc-2.34/
mkdir build
cd build/
## 这一步会出现较多报错，逐个解决
../configure  --prefix=/usr --disable-profile --enable-add-ons --with-headers=/usr/include --with-binutils=/usr/bin
# 时间会比较久，耐心等待 或者使用 # -j 8是用8个线程编译，跑的更快
make -j 8 或者 make

make install
```
通过 strings /lib64/libc.so.6 | grep -E "^GLIBC" | sort -V -r | uniq 查询是否存储在对应版本
```bash
strings /lib64/libc.so.6 | grep -E "^GLIBC" | sort -V -r | uniq
```

## 报错解决
运行到../configure --prefix=/usr/local/glibc-2.34时报错
报错1：
configure: error: in `/root/test/glibc-2.34/build’:
configure: error: no acceptable C compiler found in $PATH
```bash
yum install gcc -y
```
报错2： 
*** These critical programs are missing or too old: make bison python
*** Check the INSTALL file for required versions.
解决方案： make bison python太过老旧。 一次升级解决
### 升级GCC编译器
```bash
yum -y install centos-release-scl
yum -y install devtoolset-8-gcc devtoolset-8-gcc-c++ devtoolset-8-binutils
scl enable devtoolset-8 bash
echo "source /opt/rh/devtoolset-8/enable" >>/etc/profile
```
### 升级make
```bash
wget http://ftp.gnu.org/gnu/make/make-4.2.tar.gz
tar -xzvf make-4.2.tar.gz
cd make-4.2
./configure
make && make install
rm -rf /usr/bin/make
cp ./make /usr/bin/
make -v
```
### 升级bison
```bash
yum install -y bison
```
### 升级python
系统默认python是2.7版本，需要升级[python升级参考](/python_upgrate)