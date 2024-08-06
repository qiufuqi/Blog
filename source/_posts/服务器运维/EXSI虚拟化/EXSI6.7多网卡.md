---
title: EXSI6.7多网卡
date: 2022-10-19
tags:
  - EXSI
  - 网卡
categories: 
- 运维
- EXSI
- 网卡
keywords: 'Exsi,网卡'
cover: https://qiufuqi.github.io/img/hexo/20221019100146.png
abbrlink: exsi_wangka
comments: false
---

EXSI双网卡连接
## 检查双网卡
若服务器存在多个网卡，查看可以使用的网络适配器，链接速度有显示的，代表当前使用中的网口。
![](https://qiufuqi.github.io/img/hexo/20221019100441.png)

## 新增虚拟交换机
由于有多个物理网卡，在虚拟交换机界面择物理网卡需要新增的物理网卡，注意最开始的默认网口已经分配给虚拟机内部通信使用。
![](https://qiufuqi.github.io/img/hexo/20221019100522.png)
![](https://qiufuqi.github.io/img/hexo/20221019100554.png)

## 新增网络端口组
新增网络端口组，定义端口组的名称，这里定义为PC，绑定刚刚新建的虚拟交换机；
![](https://qiufuqi.github.io/img/hexo/20221019100623.png)

## 添加网络适配器
配置虚拟机网络适配器完成，可在每个WIN系统中添加对应的网络适配器，这样打开WIN系统就可以在电脑网络适配器看到新增的这个网口了。
![](https://qiufuqi.github.io/img/hexo/20221019100659.png)







