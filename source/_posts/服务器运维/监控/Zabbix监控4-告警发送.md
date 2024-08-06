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
abbrlink: zabbix_alert
url: zabbix_alert
comments: false
---

Zabbix预警信息发送：Email和钉钉。
# Email告警
## 媒介配置Email
管理 > 报警媒介类型，选择Email 
![](https://qiufuqi.github.io/img/hexo/20230614160152.png)

填写邮箱相关信息（我用的是126的邮箱，请根据自己邮箱设置来填写）
![](https://qiufuqi.github.io/img/hexo/20230615143029.png)

编辑消息模板，告警时会调用这里的模板信息
![](https://qiufuqi.github.io/img/hexo/20230614165349.png)

配置好相关邮箱信息后，可进行邮箱发送测试（我的邮箱成功收到邮件）
![](https://qiufuqi.github.io/img/hexo/20230614160836.png)

## 创建用户
即用户和报警媒介关联，注意创建用户要对服务器群组有读取权限才可以发送邮件
管理–>用户群组–>创建用户群组 ，添加对服务器群组读取权限
![](https://qiufuqi.github.io/img/hexo/20230615142545.png)


管理 > 用户，创建新的用户，并添加到对应用户组，添加/编辑[报警媒介]
![](https://qiufuqi.github.io/img/hexo/20230614161059.png)
编辑报警媒介，填写收件人（出现问题会给指定邮箱发送告警邮件）
![](https://qiufuqi.github.io/img/hexo/20230614161151.png)
**权限无法修改，由用户组统一分配。**


## 警报动作
配置 > 动作 > 创建动作
![](https://qiufuqi.github.io/img/hexo/20230614161435.png)
动作：输入告警名称，触发器触发的条件(测试使用，可以不选)
![](https://qiufuqi.github.io/img/hexo/20230615143653.png)
操作：
默认操作步骤持续时间改为60s，操作/回复操作 点击添加按钮 （可提前创建用户组/用户，创建用户时 要勾选报警媒介，填写接收邮件地址）
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
将被监控虚机关机，再开机，就会收到邮件提醒。
（大部分出问题再用户组权限出，要有对某个组的读写的权限）
报表 => 触发器 TOP100 可查看出现哪些问题
报表 => 动作日志 可查看是否发送邮件


# 钉钉报警
## 创建钉钉机器人
自行百度创建，拿到Webhook地址
## 升级服务器Python版本
升级Python3 [升级参考](/python_upgrate)，安装必要的组件
``` bash
[root@zabbix-server alertscripts]# wget https://bootstrap.pypa.io/pip/3.6/get-pip.py
[root@zabbix-server alertscripts]# python get-pip.py
[root@zabbix-server alertscripts]# pip install requests

```


## 创建钉钉报警脚本
``` bash
# 查看脚本存放位置
[root@zabbix-server zabbix]# grep ^AlertScriptsPath /etc/zabbix/zabbix_server.conf
AlertScriptsPath=/usr/lib/zabbix/alertscripts
[root@zabbix-server zabbix]# cd /usr/lib/zabbix/alertscripts
[root@zabbix-server zabbix]# vi dingding.py
#!/usr/bin/env python
#coding:utf-8
#zabbix钉钉报警  ./dingding.py(0) 钉钉号(1) test(2) "这个条测试信息,忽略 ID"(3) 
import requests,json,sys,os,datetime
webhook="https://oapi.dingtalk.com/robot/send?access_token=************************"
user=sys.argv[1]
text=sys.argv[3]
data={
    "msgtype": "text",
    "text": {
        "content": text
    },
    "at": {
        "atMobiles": [
            user
        ],
        "isAtAll": False
    }
}
headers = {'Content-Type': 'application/json'}
x=requests.post(url=webhook,data=json.dumps(data),headers=headers)
if os.path.exists("/var/log/zabbix/dingding.log"):
    f=open("/var/log/zabbix/dingding.log","a+")
else:
    f=open("/var/log/zabbix/dingding.log","w+")
f.write("\n"+"--"*30)
if x.json()["errcode"] == 0:
    f.write("\n"+str(datetime.datetime.now())+"    "+str(user)+"    "+"发送成功"+"\n"+str(text))
    f.close()
else:
    f.write("\n"+str(datetime.datetime.now()) + "    " + str(user) + "    " + "发送失败" + "\n" + str(text))
    print(x.json())
    f.close()
[root@zabbix-server alertscripts]# chmod +x dingding.py 
```
创建日志文件夹/文件：/var/log/zabbix/dingding.log
``` bash
[root@zabbix-server zabbix]# touch /var/log/zabbix/dingding.log
[root@zabbix-server zabbix]# chown -R zabbix.zabbix /var/log/zabbix/dingding.log
```
测试报警信息 （钉钉群中有这个手机号或者钉钉号）
``` bash
./dingding.py  185xxxxxxxx test "这个条测试信息,忽略" 
./dingding.py  钉钉号 test "这个条测试信息,忽略 ID" 
```


















