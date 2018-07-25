---
layout: post
title: OpenStack Neutron L3性能测试
description: "OpenStack Neutron L3性能测试"
tags: [OpenStack]
categories: [OpenStack]
---

###   测试概要
OpenStack Neutron L3的南北网络性能一直没有官方权威的测试，但是L3的实际承载的南北向带宽又是每一个部署OpenStack的用户最关心的问题之一。由于社区最近有了Shaker的工具，使得网络测试变得更加容易。这篇文章主要是借助官方文档来描述一下Neutron L3的南北网络性能。  

目前我们部署的OpenStack环境都是高可用的，至少有3个L3-agent，但是每个L3-agent都是SPOF。 如果L3-agent fail，则调度给该agent的所有虚拟路由器都将发生丢包，因此连接到这些虚拟路由器的所有虚拟机都将与外部网络失去通信。因此OpenStack社区实现了L3-HA的功能。  

OpenStack官网上有L3-HA的网络性能测试说明，但是L3-HA的网络拓扑是一个L3-agent连着两个虚拟路由器（master/slave）模式，但是文档中主要测试的是在路由器主备切换时候的网络丢包情况，和同时存在两个虚拟路由器时候的性能情况。但是考虑到一台虚拟路由器是slave模式，在性能测试的时候，流量并不通过salve的虚拟路由器。而且在进行带宽测试的时候并没有路由器主备切换的情况发生，因此测试结果可以作为Neutron L3的南北网络性能参考，并具有一定的参考意义。  

###    测试场景
OpenStack（Mitaka版本）高可用部署，三个Network节点（三个L3-agent），在两台不同的服务器中孵化多对虚拟机，虚拟机通过不同的子网连接到不同的路由器上，两个路由器通过Floating IP进行连接。  

![1](/images/neutron-l3-performance/1.png) 

![2](/images/neutron-l3-performance/2.jpg)   

![3](/images/neutron-l3-performance/3.jpg)  

![4](/images/neutron-l3-performance/4.jpg) 

![5](/images/neutron-l3-performance/5.jpg)  


###   测试图表TCP Download/Upload：在并发数从1-22的过程中，下载速度从700Mbits/s下降到100多Mbits/s
![10](/images/neutron-l3-performance/10.png)UDP Download/Upload：在并发数5的条件下，UDP的带宽平均值大约为765Mbits/s![11](/images/neutron-l3-performance/11.png)  在发生Restart L3的情况下，由于主备虚拟路由器的切换（master->slave），性能情况与流量直接走master虚拟路由器的性能并没有很大差别
![12](/images/neutron-l3-performance/12.jpg)  ###  测试数据：Shaker提供有关不同连接测量的最大值，最小值和平均值的统计数据。 在所有最大值中找出最大值和所有最小值中找出最小值，并且在所有平均值中统计最平均的值。 在下表中，列出了这些值。在并发数为1-22的条件下：  

![6](/images/neutron-l3-performance/6.jpg)     ![7](/images/neutron-l3-performance/7.jpg)  
### 参考文章https://docs.openstack.org/performance-docs/latest/test_plans/neutron_features/l3_ha/plan.html



