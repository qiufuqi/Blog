---
title: vCenter证书过期问题
date: 2022-12-16
tags:
  - EXSI
  - vCenter
categories: 
- 运维
- EXSI
- vCenter
keywords: 'Exsi,vCenter,证书过期'
cover: https://qiufuqi.github.io/img/hexo/20220922143047.png
abbrlink: exsi_vCenter_certify
comments: false
---

**Vmware vCenter 6.7证书过期问题**

[参考地址](https://blog.csdn.net/colalovescoffee/article/details/127528479)

# 故障现象
登陆VC报错。
![](https://qiufuqi.github.io/img/hexo/20221216171423.png)

# 进入命令行
按照报错信息，结合官方文档，判断为STS证书过期导致。
vCenter Server Appliance (VCSA) 6.5.x, 6.7.x or vCenter Server 7.0.x
**/var/log/vmware/vpxd-svcs/vpxd-svcs.log**看到类似报错:
ERROR com.vmware.vim.sso.client.impl.SecurityTokenServiceImpl$RequestResponseProcessor opId=] Server rejected the provided time range. Cause:ns0:InvalidTimeRange: The token authority rejected an issue request for TimePeriod [startTime=Thu Oct 02 09:22:13 EST 2022, endTime=Fri Oct 03 09:22:13 EST 2022] :: Signing certificate is not valid at Thu Jan 02 09:22:13 EST 2020, cert validity: TimePeriod [startTime=Wed Jan 06 20:44:39 EST 2010, endTime=Wed Jan 01 20:54:23 EST 2020]

Note: The endTime should be a date in the past if the certificate is expired.

These issue occurs when the Security Token Service (STS) certificate has expired. This causes internal services and solution users to not be able to acquire valid tokens and as a result fails to function as expected.

# 查看证书过期
(我的证书尚未过期)
``` bash
root@localhost [ ~ ]# for i in $(/usr/lib/vmware-vmafd/bin/vecs-cli store list); do echo STORE $i; sudo /usr/lib/vmware-vmafd/bin/vecs-cli entry list --store $i --text | egrep "Alias|Not After"; done
```
![](https://qiufuqi.github.io/img/hexo/20221216171652.png)
如果证书的确已经过期,继续执行以下步骤。
# 更新证书

``` bash
root@dxcvcsa [ ~ ]# /usr/lib/vmware-vmca/bin/certificate-manager
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _

                |                                                                     |

                |      *** Welcome to the vSphere 6.7 Certificate Manager  ***        |

                |                                                                     |

                |                   -- Select Operation --                            |

                |                                                                     |

                |      1. Replace Machine SSL certificate with Custom Certificate     |

                |                                                                     |

                |      2. Replace VMCA Root certificate with Custom Signing           |

                |         Certificate and replace all Certificates                    |

                |                                                                     |

                |      3. Replace Machine SSL certificate with VMCA Certificate       |

                |                                                                     |

                |      4. Regenerate a new VMCA Root Certificate and                  |

                |         replace all certificates                                    |

                |                                                                     |

                |      5. Replace Solution user certificates with                     |

                |         Custom Certificate                                          |

                |                                                                     |

                |      6. Replace Solution user certificates with VMCA certificates   |

                |                                                                     |

                |      7. Revert last performed operation by re-publishing old        |

                |         certificates                                                |

                |                                                                     |

                |      8. Reset all Certificates                                      |

                |_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _|

Note : Use Ctrl-D to exit.

Option[1 to 8]: 4     

Do you wish to generate all certificates using configuration file : Option[Y/N] ? : y

Please provide valid SSO and VC privileged user credential to perform certificate operations.

Enter username [Administrator@vsphere.local]:Administrator@vsphere.local

Enter password:

certool.cfg file exists, Do you wish to reconfigure : Option[Y/N] ? : y

Press Enter key to skip optional parameters or use Previous value.

Enter proper value for 'Country' [Previous value : US] : cn

Enter proper value for 'Name' [Previous value : CA] : CA

Enter proper value for 'Organization' [Previous value : VMware] : VMware

Enter proper value for 'OrgUnit' [Previous value : VMware Engineering] : VMware Engineering

Enter proper value for 'State' [Previous value : California] : GuangDong   

Enter proper value for 'Locality' [Previous value : Palo Alto] : Guangzhou

Enter proper value for 'IPAddress' (Provide comma separated values for multiple IP addresses) [optional] : 127.0.0.1

Enter proper value for 'Email' [Previous value : email@acme.com] : email@acme.com

Enter proper value for 'Hostname' (Provide comma separated values for multiple Hostname entries) [Enter valid Fully Qualified Domain Name(FQDN), For Example : example.domain.com] : dxcvcsa.localdns.com

Enter proper value for VMCA 'Name' :dxcVMCA

You are going to regenerate Root Certificate and all other certificates using VMCA

Continue operation : Option[Y/N] ? : y

Get site nameCompleted [Replacing Machine SSL Cert...]                 

default-site

Lookup all services

Get service default-site:45ee0951-9cf9-4c22-8641-a791f5e935c8

Don't update service default-site:45ee0951-9cf9-4c22-8641-a791f5e935c8

Get service default-site:adf34f62-1d81-467b-9f76-59304c504388

Don't update service default-site:adf34f62-1d81-467b-9f76-59304c504388

Get service default-site:452dfd21-741a-4286-b59f-e4479fd73d02

Don't update service default-site:452dfd21-741a-4286-b59f-e4479fd73d02

Get service 9356d7ff-5045-4720-a142-3e1561dc2caa

Update service 9356d7ff-5045-4720-a142-3e1561dc2caa; spec: /tmp/svcspec_o29ann0i

Get service eb760607-6057-4c8f-bffe-c4459a23361a

Update service eb760607-6057-4c8f-bffe-c4459a23361a; spec: /tmp/svcspec_f9a6t5iv

Get service e72dc500-379b-445c-a6a2-934980d7697f

Update service e72dc500-379b-445c-a6a2-934980d7697f; spec: /tmp/svcspec_q745wbdl

Get service cc66bae3-9a81-4a47-bfc2-f56b521a3491

Update service cc66bae3-9a81-4a47-bfc2-f56b521a3491; spec: /tmp/svcspec_h6wiab6b

Get service ff3c666a-8048-401c-8e5d-3cc29d783d5f

Update service ff3c666a-8048-401c-8e5d-3cc29d783d5f; spec: /tmp/svcspec_734jtjut

Get service 47bbd5fd-cdd8-4c43-a839-9cac3c4ffb14_kv

Update service 47bbd5fd-cdd8-4c43-a839-9cac3c4ffb14_kv; spec: /tmp/svcspec_5q6r0b9z

Get service 0d2020df-096e-401f-bfbe-22ab3c73e321

Update service 0d2020df-096e-401f-bfbe-22ab3c73e321; spec: /tmp/svcspec_rnepbocv

Get service 40d4c99b-3840-4e75-ae9f-01c1a1d51693

Update service 40d4c99b-3840-4e75-ae9f-01c1a1d51693; spec: /tmp/svcspec_2ej9pwvm

Get service f9210573-346b-48c1-a0f4-57e469eed937

Update service f9210573-346b-48c1-a0f4-57e469eed937; spec: /tmp/svcspec_rgu720he

Get service 18db73cb-840d-4dc9-b591-af78cb26699d

Update service 18db73cb-840d-4dc9-b591-af78cb26699d; spec: /tmp/svcspec_vhd1si6e

Get service 447163a3-d02e-41cb-bedf-6bb6bc52c882

Update service 447163a3-d02e-41cb-bedf-6bb6bc52c882; spec: /tmp/svcspec_2vt5_pkn

Get service 1f305057-ad6e-46f2-816f-b638cbe5f8cc

Update service 1f305057-ad6e-46f2-816f-b638cbe5f8cc; spec: /tmp/svcspec_ed9zzks0

Get service 47bbd5fd-cdd8-4c43-a839-9cac3c4ffb14

Update service 47bbd5fd-cdd8-4c43-a839-9cac3c4ffb14; spec: /tmp/svcspec_uu_hj1bs

Get service 81ef1813-f5da-4a52-bf5e-730b0d76c45b

Update service 81ef1813-f5da-4a52-bf5e-730b0d76c45b; spec: /tmp/svcspec_o9q1aqf5

Get service 9968f0d6-7c05-4b00-a0bf-61cd8138c29f

Update service 9968f0d6-7c05-4b00-a0bf-61cd8138c29f; spec: /tmp/svcspec_332zqona

Get service 2472164c-9862-4209-9377-e6c9310bf544

Update service 2472164c-9862-4209-9377-e6c9310bf544; spec: /tmp/svcspec_vllnxe3y

Get service e8e5ba87-5834-40e3-8697-7524754dba64

Update service e8e5ba87-5834-40e3-8697-7524754dba64; spec: /tmp/svcspec_ytjr_fpf

Get service f351ae3e-99db-4cb6-b559-2afe53406c8d

Update service f351ae3e-99db-4cb6-b559-2afe53406c8d; spec: /tmp/svcspec_ahxrtfp2

Get service 81bd2bd9-9fc1-481f-bf8f-744a54e0fb76

Update service 81bd2bd9-9fc1-481f-bf8f-744a54e0fb76; spec: /tmp/svcspec_b9p8e9r_

Get service 87a6c98a-046f-46ec-9aba-d66a30c0a91b

Update service 87a6c98a-046f-46ec-9aba-d66a30c0a91b; spec: /tmp/svcspec_l5nahdu6

Get service b496d4b6-7560-4f58-9129-ce594ee96778

Update service b496d4b6-7560-4f58-9129-ce594ee96778; spec: /tmp/svcspec_qy6458zi

Get service 3888acd4-aa58-4c5f-8b43-30f454f4d97f

Update service 3888acd4-aa58-4c5f-8b43-30f454f4d97f; spec: /tmp/svcspec_tgdq0mzy

Get service d690b63c-6105-4411-8e14-1d10259b812f

Update service d690b63c-6105-4411-8e14-1d10259b812f; spec: /tmp/svcspec_95zuwvcb

Get service 174b1a17-b44b-4967-bb94-4f7c531ba800

Update service 174b1a17-b44b-4967-bb94-4f7c531ba800; spec: /tmp/svcspec_crrn4enf

Get service 47bbd5fd-cdd8-4c43-a839-9cac3c4ffb14_authz

Update service 47bbd5fd-cdd8-4c43-a839-9cac3c4ffb14_authz; spec: /tmp/svcspec_s6zjph53

Get service 34585982-ec94-4a93-bc1f-f80eecdaf88d

Update service 34585982-ec94-4a93-bc1f-f80eecdaf88d; spec: /tmp/svcspec_p_xvj30r

Get service f8a197a6-4fdb-4dcb-baa7-cc4825f824dc

Update service f8a197a6-4fdb-4dcb-baa7-cc4825f824dc; spec: /tmp/svcspec_mnjwbgp6

Get service dfa6cc50-dbe5-4997-bd8d-949e75be87e8

Update service dfa6cc50-dbe5-4997-bd8d-949e75be87e8; spec: /tmp/svcspec_fzje6ttg

Get service eb760607-6057-4c8f-bffe-c4459a23361a_com.vmware.vsphere.client

Don't update service eb760607-6057-4c8f-bffe-c4459a23361a_com.vmware.vsphere.client

Get service bc5ba386-ce79-42de-a8f9-67c6b8f03bf1

Update service bc5ba386-ce79-42de-a8f9-67c6b8f03bf1; spec: /tmp/svcspec_40_4ncxp

Get service 024591a5-3492-4567-81d7-0439f2113196

Update service 024591a5-3492-4567-81d7-0439f2113196; spec: /tmp/svcspec__s5my1_r

Get service 5944fc2d-78d7-42f1-9a17-efc9fa0bbff3

Update service 5944fc2d-78d7-42f1-9a17-efc9fa0bbff3; spec: /tmp/svcspec_wnt0axw7

Get service eb760607-6057-4c8f-bffe-c4459a23361a_com.commvault.vsa

Don't update service eb760607-6057-4c8f-bffe-c4459a23361a_com.commvault.vsa

Updated 31 service(s)

Status : 60% Completed [Replace vpxd-extension Cert...]                    

2022-10-26T00:46:00.988Z  Updating certificate for "com.vmware.imagebuilder" extension

Status : 85% Completed [starting services...]    

Status : 100% Completed [All tasks completed successfully]  
```
更新完毕
# 查看服务状态
``` bash
service-control --stop –-all
service-control --start --all
```
![](https://qiufuqi.github.io/img/hexo/20221216171908.png)
# 查看证书状态
``` bash
root@localhost [ ~ ]# for i in $(/usr/lib/vmware-vmafd/bin/vecs-cli store list); do echo STORE $i; sudo /usr/lib/vmware-vmafd/bin/vecs-cli entry list --store $i --text | egrep "Alias|Not After"; done
```
# 登录vCenter
正常登录VC  查看证书信息 VC恢复使用
![](https://qiufuqi.github.io/img/hexo/20221216172019.png)
重新生成证书所用信息，已在证书体现，有个细节就是country填的是cn，这里显示的还是US
![](https://qiufuqi.github.io/img/hexo/20221216172100.png)
有专用脚本检测证书状态
![](https://qiufuqi.github.io/img/hexo/20221216172122.png)

# 证书存放位置
``` bash
root@dxcvcsa [ /usr/lib/vmware-vmca/share/config ]# cat   /var/tmp/vmware/certool.cfg
Country = cn
Name = CA
Organization = VMware
OrgUnit = VMware Engineering
State = GuangDong
Locality = Guangzhou
IPAddress = 127.0.0.1
Email = email@acme.com
Hostname = dxcvcsa.localdns.com
root@dxcvcsa [ /usr/lib/vmware-vmca/share/config ]#
```
**默认证书存放位置**
The Certool.cfg is located at:
vCenter Server Appliance: /usr/lib/vmware-vmca/share/config/certool.cfg
External Platform Service Controller Appliance: /usr/lib/vmware-vmca/share/config/certool.cfg
