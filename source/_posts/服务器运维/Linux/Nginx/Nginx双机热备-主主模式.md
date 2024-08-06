---
title: Nginx双机热备-主主模式
date: 2022-11-09
tags:
  - Linux
  - Nginx
  - 主主
categories: 
- 运维
- Nginx
- 双机热备-主主
keywords: 'Linux,Nginx'
description: Nginx双机热备-主主模式
cover: https://qiufuqi.github.io/img/hexo/20221109172443.png
abbrlink: nginx_zhuzhu
comments: false
---

前文已经讲述了[Nginx双机热备-主从模式](/nginx_zhucong),本文主要讲述主主模式的配置。
即前端使用两台负载均衡服务器，互为主备，且都处于活动状态，同时各自绑定一个公网虚拟IP，提供负载均衡服务；当其中一台发生故障时，另一台接管发生故障服务器的公网虚拟IP（这时由非故障机器一台负担所有的请求）。这种方案，经济实惠，非常适合于当前架构环境。

keepalived不支持跨网段ip地址
keepalived采用arp广播模式，无法跨网段。也就是10.11.7.235与10.128.2.106这种ip地址是无法通的,必须是在同一个子网内.

## 环境准备
操作系统：centos7.6
master机器(master-node)：10.11.7.231  vip：10.11.7.235
slave机器(slave-node)：10.11.7.232    vip: 10.11.7.236
slave机器(slave-node)：10.11.7.233    vip: 10.11.7.237
主主模式需要三个负载均衡的VIP:10.11.7.235 10.11.7.236 10.11.7.237

## 添加检测脚本
``` bash
# 更新vip的arp记录到网关（注意脚本中的网卡别填错了，要跟vip所在网卡一致）
[root@master-node ~]# vi /etc/keepalived/clean_arp.sh         
#!/bin/sh
VIP=$1
# 负载均衡器的公网网关地址
GATEWAY=$2                                                       
/sbin/arping -I em1 -c 5 -s $VIP $GATEWAY &>/dev/null
[root@master-node ~]# chmod 755 /etc/keepalived/clean_arp.sh
```
## keepalived配置
在主从模式的基础上修改，可参考[主从模式](/nginx_zhucong)
**master-node节点的keepalived配置,并重启keepalived服务**
master负载机上的keepalived配置：（注意，这里是双主配置，MASTER-BACKUP和BACKUP-MASTER;如果是多主，比如三主，就是MATER-BACKUP-BACKUP（101-99-99）、BACKUP-MASTER-BACKUP（99-101-97）和BACKUP-BACKUP-MASTER（97-97-101））
注意：配置中的虚拟路由标识**virtual_router_id在MASTER和BACKUP处配置不能一样**（但在主从模式下配置是一样的）
同一个virtual_router_id下 priority需设置不一样
``` bash
[root@master-node ~]# vi /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   router_id master-node
}
vrrp_script chk_http_port {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight -5
    fall 2
    rise 1 
}
vrrp_instance VI_1 {
    state MASTER
    interface ens192
    virtual_router_id 51
    priority 101
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    track_script {
        chk_http_port
    }
    virtual_ipaddress {
        10.11.7.235/24
    }
    notify_master "/etc/keepalived/clean_arp.sh 10.11.7.235 10.11.7.2"
}

vrrp_instance VI_2 {           
    state BACKUP          
    interface ens192           
    virtual_router_id 52      
    priority 99              
    advert_int 1              
    authentication {           
        auth_type PASS        
        auth_pass 1111         
    }
    track_script {                    
      chk_http_port                
    }
    virtual_ipaddress {       
      10.11.7.236/24
    }
notify_master "/etc/keepalived/clean_arp.sh 10.11.7.236  10.11.7.2"
}

vrrp_instance VI_3 {           
    state BACKUP          
    interface ens192           
    virtual_router_id 53
    priority 99           
    advert_int 1              
    authentication {           
        auth_type PASS        
        auth_pass 1111         
    }
    track_script {                    
      chk_http_port                
    }
    virtual_ipaddress {       
      10.11.7.237/24
    }
notify_master "/etc/keepalived/clean_arp.sh 10.11.7.237  10.11.7.2"
}
```
**slave-node节点的keepalived配置 backup-master-backup,并重启keepalived服务**
``` bash
! Configuration File for keepalived    
global_defs {              
  router_id slave-node                    
}
vrrp_script chk_http_port {         
    script "/etc/keepalived/check_nginx.sh"   
    interval 2                      
    weight -5                       
    fall 2                   
    rise 1                  
}
vrrp_instance VI_1 {            
    state BACKUP           
    interface ens192            
    virtual_router_id 51        
    priority 99               
    advert_int 1               
    authentication {            
        auth_type PASS         
        auth_pass 1111          
    }
    virtual_ipaddress {        
        10.11.7.235/24
    }
    track_script {                     
      chk_http_port                 
    }
  notify_master "/etc/keepalived/clean_arp.sh 10.11.7.235 10.11.7.2"
}

vrrp_instance VI_2 {           
    state MASTER          
    interface ens192           
    virtual_router_id 52      
    priority 101              
    advert_int 1              
    authentication {           
        auth_type PASS        
        auth_pass 1111         
    }
    track_script {                    
      chk_http_port                
    }
    virtual_ipaddress {       
      10.11.7.236/24
    }
notify_master "/etc/keepalived/clean_arp.sh 10.11.7.236 10.11.7.2"
}

vrrp_instance VI_3 {           
    state BACKUP          
    interface ens192           
    virtual_router_id 53
    priority 97            
    advert_int 1              
    authentication {           
        auth_type PASS        
        auth_pass 1111         
    }
    track_script {                    
      chk_http_port                
    }
    virtual_ipaddress {       
      10.11.7.237/24
    }
notify_master "/etc/keepalived/clean_arp.sh 10.11.7.237  10.11.7.2"
}
```
**slave-node节点的keepalived配置 backup-backup-master,并重启keepalived服务**
``` bash
! Configuration File for keepalived    
global_defs {              
  router_id slave-node                    
}
vrrp_script chk_http_port {         
    script "/etc/keepalived/check_nginx.sh"   
    interval 2                      
    weight -5                       
    fall 2                   
    rise 1                  
}
vrrp_instance VI_1 {            
    state BACKUP           
    interface ens192            
    virtual_router_id 51        
    priority 97              
    advert_int 1               
    authentication {            
        auth_type PASS         
        auth_pass 1111          
    }
    virtual_ipaddress {        
        10.11.7.235/24
    }
    track_script {                     
      chk_http_port                 
    }
  notify_master "/etc/keepalived/clean_arp.sh 10.11.7.235 10.11.7.2"
}

vrrp_instance VI_2 {           
    state BACKUP          
    interface ens192           
    virtual_router_id 52      
    priority 97              
    advert_int 1              
    authentication {           
        auth_type PASS        
        auth_pass 1111         
    }
    track_script {                    
      chk_http_port                
    }
    virtual_ipaddress {       
      10.11.7.236/24
    }
notify_master "/etc/keepalived/clean_arp.sh 10.11.7.236 10.11.7.2"
}

vrrp_instance VI_3 {           
    state MASTER          
    interface ens192           
    virtual_router_id 53
    priority 101             
    advert_int 1              
    authentication {           
        auth_type PASS        
        auth_pass 1111         
    }
    track_script {                    
      chk_http_port                
    }
    virtual_ipaddress {       
      10.11.7.237/24
    }
notify_master "/etc/keepalived/clean_arp.sh 10.11.7.237  10.11.7.2"
}
```
## VIP漂移验证
关闭两台负载机其中一台的keepalived服务，那么它的VIP就会自动漂移到另一台机器上。
关闭两台机器的nginx，会自动重启（前提是keepalived服务要启动）！对网站域名的访问丝毫不受影响。