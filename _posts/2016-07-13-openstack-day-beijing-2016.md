---
layout: post
title: OpenStack Day China Beijing 2016
description: "OpenStack Day China Beijing 2016"
tags: [OpenStack]
categories: [OpenStack]
---


![image](/images/openstack-beijing-2016/1.JPG)  


####  注:由于会场太多，只总结了常见的成熟的新兴的技术及场景以及OpenStack的一些趋势  


系统整体可靠性是否足以满足企业级业务要求


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


![image](/images/openstack-beijing-2016/10.JPG)  





![image](/images/openstack-beijing-2016/14.png)





![image](/images/openstack-beijing-2016/15.png)

在SDN这一块没有看到很多的解决方案。只看到了华为的dragonflow sdn controller


![image](/images/openstack-beijing-2016/16.png)



![image](/images/openstack-beijing-2016/2.png)





解决方案  

* 升级前备份，保证出现异常时可以Rollback
* 数据面保持一致－ovsagent重启
* 使用mysqlsync同步备份数据库
* 云平台控制面服务短暂中断10-20分钟
* 服务启动，功能验证



ceph是本次大会出场率最高的存储方案了  

ceph特点    

支持多种标准存储接口(如s3)    


* Cobber装机
* Zabbix监控
* Ansible批量运维
* Ansible用户管理
* ELK日志管理


### ansible特点  

####  维护特点
* 多重API接口


* Openipmi支持
* 初始化网络信息









### Disadvantage

* Docker is green(太新了)
* Additional complexity(需要运维人员学习docker)
有一些厂商根据Murano开发了自己的应用商店，以对接OpenStack


![image](/images/openstack-beijing-2016/9.JPG)


##  趋势  

全分布式虚拟网络  

![image](/images/openstack-beijing-2016/7.png)

计算节点高可用  


![image](/images/openstack-beijing-2016/12.png)

## 一些经验总结  

![image](/images/openstack-beijing-2016/11.jpg)




### 客户端浏览器到容器service时无法获取client ip  


### Dashboard主机VNC控制台打不开

nova-api接收到获取VNC请求以后，发送消息给nova－compute进程，nova－compute发送回应消息，但是由于nova－api接受响应消息丢失了，所以找不到exchange，出现通信故障，同时其nova－consoleauth也出现了连接丢失，重启nova-api以及nova－consoleauth以后重建连接功能正常。OpenStack社区的默认配置没有开启心跳功能，所以服务进程和rabbitMQ的连接在某种情况下会丢失

###  OpenStack Compute节点不定期变化为不可用或显示出一些未部署的计算节点，导致主机相关功能不可用
容器化OpenStack Compute节点默认host配置为节点的hostname，而Docker容器的hostname为容器ID，当容器故障恢复或重启后，其容器ID及hostname会发生变化，引起服务问题