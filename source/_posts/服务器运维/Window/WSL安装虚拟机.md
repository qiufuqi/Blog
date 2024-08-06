---
title: Win11菜单切换
date: 2022-09-1
tags:
  - Windown
  - win11菜单
categories: 
- Windown
- win11菜单
keywords: 'Win11菜单,切换'
description: Win11菜单切换
cover: https://qiufuqi.github.io/img/hexo/20231205135201.png
abbrlink: win11_caidan
comments: false
top: 60
---

安装WSL虚拟机
## 寻找可提供的源
输入wsl --list --online 或者执行wsl -l -o
管理员模式下运行
``` bash
PS C:\Users\Austines> wsl -l -o
以下是可安装的有效分发的列表。
请使用“wsl --install -d <分发>”安装。

NAME            FRIENDLY NAME
Ubuntu          Ubuntu
Debian          Debian GNU/Linux
kali-linux      Kali Linux Rolling
openSUSE-42     openSUSE Leap 42
SLES-12         SUSE Linux Enterprise Server v12
Ubuntu-16.04    Ubuntu 16.04 LTS
Ubuntu-18.04    Ubuntu 18.04 LTS
Ubuntu-20.04    Ubuntu 20.04 LTS
```

## 安装指定源
``` bash
PS C:\Users\Austines> wsl --install -d Ubuntu-20.04
正在下载: Ubuntu 20.04 LTS
[==========================99.3%========================== ]
```