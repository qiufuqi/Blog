---
title: Linux生成密钥
date: 2024-8-14
tags:
  - Linux
  - 密钥
categories: 
- 运维
- Linux
- 密钥
keywords: 'Linux,密钥'
cover: https://qiufuqi.github.io/img/hexo/20230327084004.png
abbrlink: limux_miyao
url: limux_miyao
comments: false
---

使用ssh-keygen命令 生成文件路径：~/.ssh/
``` bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```
- -t rsa: 指定密钥类型为RSA。
- -b 4096: 指定密钥长度为4096位，以提高安全性。
- -C "your_email@example.com": 添加注释，通常用于记录您的电子邮件地址。

如果需要一个PEM格式的密钥，你可以使用openssl工具来生成。如果你只是想要一个OpenSSH格式的密钥，并且希望私钥文件具有.pem扩展名，你可以使用ssh-keygen工具生成，并手动更改文件扩展名。

如果你需要生成一个OpenSSH格式的密钥，并希望私钥文件名为id_rsa.pem：
使用ssh-keygen命令生成密钥对，并指定私钥文件名为id_rsa.pem：
``` bash
ssh-keygen -t rsa -b 4096 -C "your-email@example.com" -f ~/.ssh/id_rsa.pem
```
- -f ~/.ssh/id_rsa.pem 指定了私钥文件的路径和文件名。
运行上述命令后，你将会得到两个文件：
~/.ssh/id_rsa.pem：这是私钥文件。
~/.ssh/id_rsa.pem.pub：这是对应的公钥文件。