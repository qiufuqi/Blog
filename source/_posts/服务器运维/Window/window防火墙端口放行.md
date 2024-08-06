---
title: window防火墙端口放行
date: 2022-11-21
tags:
  - Windown
  - 端口
categories: 
- Windown
- 放行端口
keywords: 'window,端口'
cover: https://qiufuqi.github.io/img/hexo/20221121084243.png
abbrlink: window_firewalld
comments: false
---

**Windows Server启用防火墙并开放80端口**
Windows Server是自带防火墙的，可以过滤一些包。一般情况下为了安全都要打开防火墙，可以防止一些攻击行为。同时，由于启用防火墙后会封闭所有的端口，计算机根本就不会接受外部对端口发起的连接请求，也就无法对外提供服务。
如果要端口开放,必须要有进程程序安装对应该端口对应，比如安装web后对应80端口；同时服务器内部防火墙对端口不要做限制，或者需要添加例外。

## 防火墙高级设置
控制面板 => Windows防火墙 => 高级设置(左侧)
![](https://qiufuqi.github.io/img/hexo/20221121084445.png)
## 新建规则
入站规则(右击) => 新建规则
![](https://qiufuqi.github.io/img/hexo/20221121084619.png)
## 选择规则类型
规则类型 选择“端口”
![](https://qiufuqi.github.io/img/hexo/20221121084656.png)
## 输入端口号
输入要开启的端口号，例如“80“点击“下一步”
![](https://qiufuqi.github.io/img/hexo/20221121085200.png)
选择“允许连接”，点“下一步”；可按默认选中“域”“专用”“公用”
![](https://qiufuqi.github.io/img/hexo/20221121085254.png)
![](https://qiufuqi.github.io/img/hexo/20221121085303.png)
## 输入规则名称
输入名称和描述，点击完成
![](https://qiufuqi.github.io/img/hexo/20221121085339.png)
## 查看规则
可看到自己创建的入站规则
![](https://qiufuqi.github.io/img/hexo/20221121085401.png)

完成后，服务器中的防火墙对网页上的80端口就不再阻拦，输入服务器地址，连接vpn后，外部网络也可访问服务器的项目。












