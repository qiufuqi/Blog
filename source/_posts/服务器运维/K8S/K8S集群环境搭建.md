---
title: K8S集群环境搭建
date: 2022-09-16
tags:
  - K8S
  - 集群
categories: 
- 运维
- K8S
- 环境
keywords: 'K8S,集群'
description: K8S集群环境搭建
cover: https://qiufuqi.github.io/img/hexo/20220915145759.png
abbrlink: k8s_install
url: k8s_install
comments: false
---

Kuboard-Spray 是一款可以在图形界面引导下完成 Kubernetes 高可用集群离线安装的工具。
- 开源仓库的地址：[Kuboard-Spray](https://github.com/eip-work/kuboard-spray)
- 环境安装地址为：[安装参考地址](https://www.kuboard.cn/install/install-k8s.html)

提前关闭 SELinux 以及 防火墙。

# 安装Kuboard-Spray
服务器（10.128.1.61）的最低要求为：
- 1核2G
- 不少于 10G 磁盘空余空间
- 已经安装好 docker

执行的命令如下：
``` bash
[root@localhost ~]# docker run -d \
  --privileged \
  --restart=unless-stopped \
  --name=kuboard-spray \
  -p 80:80/tcp \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v ~/kuboard-spray-data:/data \
  eipwork/kuboard-spray:latest-amd64
  # 如果抓不到这个镜像，可以尝试一下这个备用地址：
  # swr.cn-east-2.myhuaweicloud.com/kuboard/kuboard-spray:latest-amd64
```
持久化
- KuboardSpray 的信息保存在容器的 /data 路径，请将其映射到一个您认为安全的地方，上面的命令中，将其映射到了 ~/kuboard-spray-data 路径；
- 只要此路径的内容不受损坏，重启、升级、重新安装 Kuboard-Spray，或者将数据及 Kuboard-Spray 迁移到另外一台机器上，您都可以找回到原来的信息。

在浏览器打开地址 http://这台机器的IP，输入用户名 admin，默认密码 Kuboard123，即可登录 Kuboard-Spray 界面


# 加载离线资源包
在 Kuboard-Spray 界面中，导航到 **系统设置 --> 资源包管理** 界面，可以看到已经等候您多时的 Kuboard-Spray 离线资源包，如下图所示：
![](https://qiufuqi.github.io/img/hexo/20220915140550.png)
点击 导 入 按钮，在界面的引导下完成资源包的加载。
![](https://qiufuqi.github.io/img/hexo/20220915140851.png)
等待资源包安装成功。

# 规划并安装集群
在 Kuboard-Spray 界面中，导航到 **集群管理** 界面，点击界面中的 **添加集群安装计划** 按钮，填写表单如下：
- 集群名称： 自定义名称，本文中填写为 Yurun，此名称不可以修改；
- 资源包：选择前面步骤中导入的离线资源包。
![](https://qiufuqi.github.io/img/hexo/20220915141432.png)


点击上图对话框中的 确定 按钮后，将进入集群规划页面，在该界面中添加您每个集群节点的连接参数并设置节点的角色，如下图所示：
***kuboard-spray 所在机器不能当做 K8S 集群的一个节点，因为安装过程中会重启集群节点的容器引擎，这会导致 kuboard-spray 被重启掉。***
![](https://qiufuqi.github.io/img/hexo/20220915144748.png)
- 最少的节点数量是 1 个；
- ETCD 节点、控制节点的总数量必须为奇数；
- 在全局设置 标签页，可以设置节点的通用连接参数，例如所有的节点都使用相同的 ssh 端口、用户名、密码，则共同的参数只在此处设置即可；
- 在节点标签页，如果该节点的角色包含 etcd 则必须填写 **ETCD 成员名称** 这个字段；

点击上图的 **保存** 按钮，再点击 **安装设置集群** 按钮，可以启动集群的离线安装过程，如下图所示：
![](https://qiufuqi.github.io/img/hexo/20220915145140.png)

耐心等待，时间可能久一点(我重复安装了几次)。成功时将显示如下页面：
![](https://qiufuqi.github.io/img/hexo/20220916085215.png)

集群日志界面提示您集群已经安装成功，此时您可以返回到集群规划页面，此界面将自动切换到 访问集群 标签页，如下图所示：

界面给出了三种方式可以访问 kubernetes 集群：

在集群主节点上执行 kubectl 命令
获取集群的 .kubeconfig 文件
将集群导入到 kuboard管理界面

# 安装多集群管理工具
Kuboard v3 [kuboard v3](https://www.kuboard.cn/install/v3/install-in-k8s.html#%E6%96%B9%E6%B3%95%E4%B8%80-%E4%BD%BF%E7%94%A8-hostpath-%E6%8F%90%E4%BE%9B%E6%8C%81%E4%B9%85%E5%8C%96)
``` bash
# 在10.128.1.62执行如下命令
kubectl apply -f https://addons.kuboard.cn/kuboard/kuboard-v3.yaml
# 您也可以使用下面的指令，唯一的区别是，该指令使用华为云的镜像仓库替代 docker hub 分发 Kuboard 所需要的镜像
# kubectl apply -f https://addons.kuboard.cn/kuboard/kuboard-v3-swr.yaml

# 执行指令 watch kubectl get pods -n kuboard，等待 kuboard 名称空间中所有的 Pod 就绪
[root@node1 ~]# kubectl get pods -n kuboard
NAME                          READY   STATUS              RESTARTS      AGE
kuboard-etcd-bgzfk            0/1     ContainerCreating   0             4m25s
kuboard-etcd-p87ds            0/1     ContainerCreating   0             4m25s
kuboard-etcd-x5rm2            1/1     Running             2 (13s ago)   4m25s
kuboard-v3-84f9bf8bfc-5mwh5   0/1     ContainerCreating   0             4m25s
kuboard-v3-node3              1/1     Running             0             5h30m

```
http://10.128.1.62:30080
默认用户名: admin
默认密 码: Kuboard123





