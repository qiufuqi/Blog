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

恢复Win10右键菜单的方法：
``` bash
1.Win+R运行CMD
2.输入：reg add HKCU\Software\Classes\CLSID\{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}\InprocServer32 /f /ve
3.重启电脑​
```