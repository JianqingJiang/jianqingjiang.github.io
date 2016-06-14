---
layout: post
title: 裸机多节点自动化OpenStack部署
description: "裸机多节点自动化OpenStack部署"
tags: [OpenStack]
categories: [OpenStack]
---

###    镜像制作
按照步骤完成OpenStack部署。加入一块硬盘，使用Linux dd命令把OpenStack的节点拷贝到硬盘中，可用gzip压缩，利用liveCD和分区对拷的方式使裸机启动，利用lvm进行硬盘调整。

###    自动化
开发shell脚本（dialog）实现自动化  
![curl](/images/openstack_bare_mutinode_install/1.png)  
![curl](/images/openstack_bare_mutinode_install/2.png)  
![curl](/images/openstack_bare_mutinode_install/3.png)  
![curl](/images/openstack_bare_mutinode_install/4.png)  
![curl](/images/openstack_bare_mutinode_install/5.png)  
ps:此方案经验证  