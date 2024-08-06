---
title: Zabbix监控4-告警发送
date: 2023-6-14
tags:
  - Zabbix
  - agent
  - 告警
categories: 
- 运维
- 监控
- 告警
keywords: 'Zabbix,监控,Agent,告警'
cover: https://qiufuqi.github.io/img/hexo/20221116102126.png
abbrlink: zabbix_show
url: zabbix_show
comments: false
---

Zabbix预警信息发送：Email和钉钉。
# Email告警
## 媒介配置Email
管理 > 报警媒介类型，选择Email 
![](https://qiufuqi.github.io/img/hexo/20230614160152.png)

填写邮箱相关信息（我用的是126的邮箱，请根据自己邮箱设置来填写）
![](https://qiufuqi.github.io/img/hexo/20230614160721.png)
编辑消息模板，告警时会调用这里的模板信息
![](https://qiufuqi.github.io/img/hexo/20230614165349.png)

配置好相关邮箱信息后，可进行邮箱发送测试（我的邮箱成功收到邮件）
![](https://qiufuqi.github.io/img/hexo/20230614160836.png)

## 收件人配置
管理 > 用户，选择Zabbix用户编辑，添加/编辑[报警媒介]
![](https://qiufuqi.github.io/img/hexo/20230614161059.png)
编辑报警媒介，填写收件人（出现问题会给指定邮箱发送告警邮件）
![](https://qiufuqi.github.io/img/hexo/20230614161151.png)

## 警报动作
配置 > 动作 > 创建动作
![](https://qiufuqi.github.io/img/hexo/20230614161435.png)
输入告警名称，默认操作步骤持续时间改为60s，操作/回复操作 点击添加按钮 （可提前创建用户组/用户，创建用户时 要勾选报警媒介，填写接收邮件地址）
发送消息可以按照个人，也可以按照组来发送，消息内容自定义。

故障操作：
![](https://qiufuqi.github.io/img/hexo/20230614163020.png)
自定义消息内容 或者使用报警媒介那边的消息模板
``` bash
主题：Host:{HOST.NAME}: {TRIGGER.NAME}故障
消息：
{Zabbix告警：
Warning 主机:{HOST.NAME}
Host:{HOST.IP}
监控项目:{ITEM.NAME}
监控取值:{ITEM.LASTVALUE}
报警等级:{TRIGGER.SEVERITY}
当前状态:{TRIGGER.STATUS}
报警信息:{TRIGGER.NAME}
报警data:{EVENT.DATE} {EVENT.TIME}
事件ID:{EVENT.ID}
}
```

恢复操作：
![](https://qiufuqi.github.io/img/hexo/20230614162602.png)
自定义消息内容
``` bash
主题：Host:{HOST.NAME}: {TRIGGER.NAME}已恢复!
消息：
{Zabbix恢复：
Warning 主机:{HOST.NAME}
Host:{HOST.IP}
监控项目:{ITEM.NAME}
监控取值:{ITEM.LASTVALUE}
报警等级:{TRIGGER.SEVERITY}
当前状态:{TRIGGER.STATUS}
报警信息:{TRIGGER.NAME}
报警data:{EVENT.DATE} {EVENT.TIME}
恢复时间:{EVENT.RECOVERY.DATE} {EVENT.RECOVERY.TIME}
持续时间:{EVENT.AGE}
事件ID:{EVENT.ID}
}
```
## 验证报警





















