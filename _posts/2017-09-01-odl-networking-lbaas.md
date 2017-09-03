---
layout: post
title: ODL-Networking之	LBaaS实现
description: "ODL-Networking之LBaaS实现"
tags: [OpenStack]
categories: [OpenStack]
---

###   LBaaS v1与v2区别
LBaaS v1: introduced in Juno (deprecated in Liberty)（lbaasv1在liberty版本开始被废除）  
LBaaS v2: introduced in Kilo（v2在kilo版本开始使用）  

两种方式都是使用了agents，这个agent获取HAProxy的配置然后处理HAProxy的damon。  
LBaaS v2增加了listeners的功能。LBaaS v2允许在一个load balancer的IP上配置多个listener的port  
另一种实现方式是名为Octavia，它用独立的API和线程，通过在compute服务生成的虚拟机中来创建load balancer，而且不需要为Octavia创建agent  

目前没有lbaas从v1迁移到v2的机制，如果选择从v1迁移到v2，那么据必须重新创建loadblancers,pools和health monitors。  


这是loadbalancer的新的架构图  

![image](/images/odl-networking-lbaas/1.png)

###   Load balancer
load balancer占用一个neutron network的port，然后将将一个IP绑在load balancer上  

###   Listener
Load balancers可以被从多个port上获取requests。每一个port被一个listener监听  

###   Pool
一个pool中有多个Member  

###   Member
Member就是具体接收流量的虚拟机  

###   Health monitor
Members虚拟机可能会时不时的下线，monitor的工作是将流量引导到健康的member上去。Health monitor与pools绑定在一起。  

###   LBaaS v2
LBaaS v2 可以通过service plug-ins实现多种操作方式。两种常用的方式是通过agent或者Octavia services. 两种实现方式都采用了 LBaaS v2 API.  

###  配置
配置可以参考
https://docs.openstack.org/mitaka/networking-guide/config-lbaas.html
git checkout stable/ocata  
yum install -y  openstack-neutron-lbaas  

![image](/images/odl-networking-lbaas/2.png)

###  注意点
my configuration is as follows:
/etc/neutron/neutron.conf:
service_plugins=networking_odl.l3.l3_odl.OpenDaylightL3RouterPlugin,neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPluginv2
/etc/neutron/neutron_lbaas.conf:
service_provider=LOADBALANCERV2:opendaylight:networking_odl.lbaas.driver_v2.OpenDaylightLbaasDriverV2:default
Then I run the command "service neutron-server restart ",


###  测试


![image](/images/odl-networking-lbaas/3.png)

![image](/images/odl-networking-lbaas/4.png)
