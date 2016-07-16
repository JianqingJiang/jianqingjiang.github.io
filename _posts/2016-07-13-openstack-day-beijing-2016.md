---
layout: post
title: OpenStack Day China Beijing 2016
description: "OpenStack Day China Beijing 2016"
tags: [OpenStack]
categories: [OpenStack]
---


![image](/images/openstack-beijing-2016/1.JPG)  


####  注:由于会场太多，只总结了常见的成熟的新兴的技术及场景以及OpenStack的一些趋势  ## OpenStack 目前面临的挑战
####  系统高可靠性

系统整体可靠性是否足以满足企业级业务要求
####  资源多样性能否根据业务的要求提高多种不同的资源类型

####  API开放性

能否开发标准的OpenStack API以满足客户云管理工具对接的需求

####  运维自动化能力

能否实现高水平的自动化运维以降低系统运行成本

####  水平可扩展性

能否实现系统规模跟随业务总量发展的水平扩容

####  业务高并发性

系统性能是否足以满足高并发性业务的要求

####  兼容性
能否兼容企业的原有的VMWare和其他的厂商的系统


> 人正在成为云计算时代的最大瓶颈  


![image](/images/openstack-beijing-2016/10.JPG)  我们需要的是类似iPhone的设计思想，不论小孩和老人，一上手就可以使用，实现完全的自动化，提供简单易懂的界面，除去华而不实的功能，并在设计背后提供高可用，高稳定性。
--- 
## Suse 
70%－80%银行在使用Suse的操作系统，Suse在这次会上可以看得出来投了很多精力（钱），Suse提出的解决方案，OpenStack层面我觉得就是just fine，但是他们靠操作系统占了优势
* 内核在线热补丁－“零停机时间”

![image](/images/openstack-beijing-2016/14.png)

* 内存热插拔实现机制
![image](/images/openstack-beijing-2016/3.png)
* CPU热插拔场景  

![image](/images/openstack-beijing-2016/15.png)## SDN

在SDN这一块没有看到很多的解决方案。只看到了华为的dragonflow sdn controller
DragonFlow

![image](/images/openstack-beijing-2016/16.png)## Scalability(扩展性)
在大规模部署HA OpenStack中会出现很严重的数据库同步问题，在数百台机器中，在很长的网络收敛速度下，数据库会变成很严重的瓶颈，因此，常用部署多个OpenStack来代替部署一个OpenStack的方式来解决  
DB Consistency(数据库同步)

![image](/images/openstack-beijing-2016/2.png)
## OpenStack升级
目前升级存在的问题  
* 升级步骤多，操作琐细、手动效率低，容易出错* 手动升级导致更长的云平台控制面停服* 手动升级无法确保多套环境的一致性操作* 升级操作审计不好做，问题回溯也比较困难


解决方案  

* 升级前备份，保证出现异常时可以Rollback* 各组件代码升级、配置升级、数据库升级
* 数据面保持一致－ovsagent重启
* 使用mysqlsync同步备份数据库
* 云平台控制面服务短暂中断10-20分钟
* 服务启动，功能验证
## Ceph


ceph是本次大会出场率最高的存储方案了  

ceph特点    
高扩展性：使用普通x86服务器，支持TB到PB级的扩展   高可靠性：没有单点故障，多数据副本，自动管理，自动修复  高性能：数据分布均衡，并行化度高，不需要元数据服务器    
支持多种标准存储接口(如s3)    有一些厂商还使用了ceph做存储的备份，其中这种备份系统还比较有趣    
![image](/images/openstack-beijing-2016/8.png)## 自动化运维

* Cobber装机
* Zabbix监控
* Ansible批量运维
* Ansible用户管理
* ELK日志管理


### ansible特点  #### 批量管理* 配置管理、自动升级* 常规运维巡检* 数据周期备份####  部署能力* 跨平台支持* 模块化部署* 丰富的编排能力

####  维护特点* 易读的语法(YAML)，简单易于操作* 内嵌丰富常用模块
* 多重API接口* 通过ssh连接远处主机无须安装任何依赖###   Cobbler特点
####  多操作系统支持* 支持CentOS/Redhat* 支持Ubuntu/Debian* 支持Esxi 5* 支持FreeBSD* 支持XenServer
#### IPMI管理
* Openipmi支持* 启动停止／关闭／带外管理* Ks文件灵活编排
* 初始化网络信息* 初始化系统参数* 分发自动控制
###  Crowbar+Chef(Suse)特点  
* 硬件发现* 裸机管理、安装* Firmware更新* 服务安装配置![image](/images/openstack-beijing-2016/13.png)##  Kolla
Kolla场很火爆，容器化部署OpenStack几乎已经是势不可挡，海云捷迅下一个产品就是完全使用容器来部署OpenStack现在的Kolla项目已经可以实现数十台的高可用部署。当然社区也在开发使用kubernates来管理容器（目前是使用ansible）
### Feature
* All active high availability* Ceph backend storage* Support multi Linux distro(CentOS/OracleLinux/Ubuntu)* Build from package or build form source* Small runtime dependency footprint,only need docker-py and docker-engine* Docker container for atomic upgrades
###  Implementation
* Use Dockerfile + jinja2 to build image* Use image dependency(build faster and smaller size)* Ansible-palybooks as deployment tool* Containerized everything(libvirt/openvswitch/neutron)* Each container has only one process* Use host network* Better configuration management


### Disadvantage

* Docker is green(太新了)
* Additional complexity(需要运维人员学习docker)##  Murano
有一些厂商根据Murano开发了自己的应用商店，以对接OpenStack


![image](/images/openstack-beijing-2016/9.JPG)


##  趋势  

全分布式虚拟网络  

![image](/images/openstack-beijing-2016/7.png)

计算节点高可用  


![image](/images/openstack-beijing-2016/12.png)

## 一些经验总结  

![image](/images/openstack-beijing-2016/11.jpg)


## 容器化OpenStack后出现的一些问题：
 
### 客户端浏览器到容器service时无法获取client ip  
web应用通过tcp请求源地址或http扩展头x-forwarded-for中请求链中的client ip来辨别客户端IP地址。但在kubernates环境下进行service ip到container ip的转换时会对tcp连接请求的源目地址进行替换，docker容器内的web应用获取的remote address即为转换后的docker0网关地址。而当客户端浏览器到容器service之间无任何http代理或反向代理未开启x-forwarded-for扩展头扩展时，http header中不包含x-forwarded-for头，无法获取client ip

### Dashboard主机VNC控制台打不开

nova-api接收到获取VNC请求以后，发送消息给nova－compute进程，nova－compute发送回应消息，但是由于nova－api接受响应消息丢失了，所以找不到exchange，出现通信故障，同时其nova－consoleauth也出现了连接丢失，重启nova-api以及nova－consoleauth以后重建连接功能正常。OpenStack社区的默认配置没有开启心跳功能，所以服务进程和rabbitMQ的连接在某种情况下会丢失

###  OpenStack Compute节点不定期变化为不可用或显示出一些未部署的计算节点，导致主机相关功能不可用
容器化OpenStack Compute节点默认host配置为节点的hostname，而Docker容器的hostname为容器ID，当容器故障恢复或重启后，其容器ID及hostname会发生变化，引起服务问题