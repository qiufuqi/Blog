---
title: Linux免密登录
date: 2023-4-18
tags:
  - Linux
  - CentOS
  - 免密
categories: 
- 运维
- 免密
keywords: 'Linux,CentOS,免密'
cover: https://qiufuqi.github.io/img/hexo/20230418114913.png
abbrlink: centos_ssh
comments: false
---

免密登录命令：
``` bash
# 1.进入.ssh目录：  
cd ~/.ssh
# 2.生成一对密钥： 
ssh-keygen -t rsa
# 3.发送公钥：
ssh-copy-id 192.168.xx.xxx
# 4.免密登录测试： 
ssh 192.168.xx.xxx
```
# 免密登录原理
免密登录原理通过RSA公开密钥算法的一种应用。RSA是公开密钥密码体制的一种使用不同的加密密钥与解密密钥，“由已知加密密钥推导出解密密钥在计算上是不可行的”密码体制(非对称加密) 。在公开密钥密码体制中，加密密钥（即公开密钥）PK是公开信息，而解密密钥（即秘密密钥）SK是需要保密的。加密算法E和解密算法D也都是公开的。虽然解密密钥SK是由公开密钥PK决定的。

**通俗的来说就是同时生成公钥和私钥，私钥自己保存，公钥发给其他人。**

# 配置ssh
进入.ssh目录，生成密钥ssh-keygen -t rsa
``` bash
[root@localhost ~]# cd .ssh/

#生成一对密钥，使用rsa通用密钥算法，这时需要有三次回车；
[root@localhost .ssh]# ssh-keygen -t rsa
[root@localhost .ssh]# ls
authorized_keys  id_rsa  id_rsa.pub  known_hosts
```
| Align `left`   | center align |   Align right |
| :------------- | :----------: | ------------: |
| `left`-aligned |   centered   | right-aligned |
| `左`对齐        |    中对齐     |         右对齐 |

| Align `left`   | center align |
| :------------- | :---------- |
| known_hosts |   记录ssh访问过计算机的公钥（public key）|
| id_rsa |   生成的私钥|
| id_rsa.pub |   生成的公钥|
| authorized_keys |   存放授权过的无密登录服务器公钥（后面会提到）|

# 发送公钥
将公钥发送给目标IP(需要输入密码)，目标IP则可以免密码登录 
命令：ssh-copy-id 10.11.8.63
命令：ssh-copy-id -i .ssh/id_dsa.pub root@10.11.8.63
``` bash
[root@localhost .ssh]# ssh-copy-id 10.11.8.63

# 后期就可以直接登录
[root@localhost .ssh]# ssh 10.11.8.63
Last login: Tue Apr 18 14:14:41 2023 from 10.11.8.68
[root@devqiu ~]# 
```