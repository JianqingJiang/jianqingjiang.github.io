---
layout: post
title: OpenStack巴塞罗那峰会Day3
description: "OpenStack巴塞罗那峰会Day3"
tags: [OpenStack]
categories: [OpenStack]
---

##  简述

第三、四天的峰会基本上厂商的Presentation都差不多结束了，重点都放在Design Summit上面了。这样也提供了很好的机会跟开发者面对面进行交流。因为有些议题比较庞大，所以今天还是有一些分享，比如热门的老牌项目Nova以及Neutron,这两个项目基本上改一个的话另一个也需要改动。在新兴的项目Docker,k8s等跟OpenStack进行结合，但是因为这些项目讨论的余地还是比较大的，所以到了第三天仍有不少的Presentation。由于Neutron这个项目的特殊性，基本上需要跟很多项目挂钩，比如与容器的网络对接，与上层的网络管理的项目的API等对接。网络除了功能以外还有性能的问题，一些厂商也提供了一些提高性能的解决方案。

##  1.虚拟机疏散

虚拟机的疏散问题很早之前就已经被反复讨论了，这次峰会也不例外。虚拟机疏散就是虚拟机跑在Hypervisor上，如果Hypervisor挂掉之后需要将虚拟机疏散到正常的compute节点之上。这个问题看似简单，这次峰会很多discussion都是关于这个问题或者是提到了这个问题。因为讲虚拟机疏散这个指令发起之后，调度器就要检查有没有多余的硬盘资源，计算资源以及网络资源。需要三个资源都被确认OK之后，才可以被转移。但是由于OpenStack系统是动态变化的，当三块资源被确认之后，如果用户又创建了一些虚拟机，那么就可能导致疏散的虚拟机得不到某一个足够的资源，从而导致迁移失败。另一个重要的问题时社区在讨论疏散的“undo”功能，即一旦疏散失败或者之前宕掉的compute节点恢复之后怎么把疏散的虚拟机由重新迁移回来。以上说的都是比较粗略的，具体涉及到迁移的虚拟机的mac地址是否需要变化，在混合云的迁移的不同的hypervisor等情况还是很难解决的。




##  2.提高Neutron网络的扩展性

由于现在的Neutron网络都是虚拟机连在虚拟子网或者vlan子网上，子网与子网之间通过vxlan或者gre等三层网络技术进行连接的，但是这样的网络架构对于网络的扩展性来说却是不利的，当网络上了规模之后Neutron的瓶颈就出现了。基本上目前使用社区版的网络，高可用部署，VRRP的网络冗余方案只能支持数百台服务器的规模，当OpenStack的规模到了很大的程度，则必须想到对策解决扩展性的问题。  

![image](/images/openstack-barcelona-summit-3/1.jpg)  

IBM的一个silde就提到传统neutron的问题  
These network become complex at largr scale  
Also have large failure domains  

就是在二层的网络中再加上一个segment头部标志，把每一个物理机架作为一个segment，这样网络的规模变的非常大的时候，对每一个机架的segmenet都可以进行很好的管理，同一个segment的网络是具有路由功能的，而不是之前的网络的单纯桥接方式。    

![image](/images/openstack-barcelona-summit-3/2.jpg)

他们提出了新的网络架构，现在已经被merge到Newton版本中了  

![image](/images/openstack-barcelona-summit-3/3.jpg)

##   3.Dragonflow

华为的dragonflow团队正在研究BGP的问题，以及跨数据中心的网络控制，其中的mac地址变换，以及虚拟机从一个数据中心迁移到另一个数据中心等问题进行了研究，以及对这个虚拟机配置的security group在迁移到另一个数据中心是否生效等问题。在控制不同的数据中心时候对不同的数据中心中的每个虚拟机都需要知道怎么路由或者进行l2的转发，在最后一天的discussion中针对以上的问题进行了讨论。  

![image](/images/openstack-barcelona-summit-3/4.jpg)

在Okata版本中的计划如下：  

```
PLAN:
Deployment
Kolla
Ansible +1+1
Puppet +1+1
Troubleshooting Tools
North Bound database changes
subnets to have their own table, and not part of lswitch (Networks)+1
IPv6+1
SFC+1+1
Distributed Load Balancing+1
BGP advertising for dynamic routing +1+1+1+1
Distributed DNS+1
Multicast and IGMP
Is there a viable use case for multicast in the overlay network?
Backend Drivers
eBPF+1
P4
Migration from ML2/OVS ?
Dragonflow API Layer 
TAP as a Service
VLAN Aware VMs
Distributed NAT+1
```
在第三天的discussion中把dragonflow的测试报告也公布了出来    

![image](/images/openstack-barcelona-summit-3/8.jpg)

![image](/images/openstack-barcelona-summit-3/9.jpg)

![image](/images/openstack-barcelona-summit-3/10.jpg)

对于Neutron来说则是有很大提升的，dragonflow也是目前做对接OpenStack做的最好的。我司也在做一个对接OpenStack的控制器，名称为[DCFabric](http://launchpad.net/dcfabric "DCFabric")  

##    4.Okata Road Map

Okata版本基本上的roadmap也出来了，社区做了一个session是讲各个项目的下个版本的计划  

![image](/images/openstack-barcelona-summit-3/5.jpg)  

![image](/images/openstack-barcelona-summit-3/6.jpg)  

![image](/images/openstack-barcelona-summit-3/7.jpg)  

由于我比较关注网络方面，因此大部分都是网络。其实4天听下来网络的问题是最多的，加入向Docker等虚拟化技术对Nova的影响远远没有网络影响大，而且在规模大了之后，网络的延迟的吞吐量也是很值得关注的，而且Neutron作为OpenStack必须部署的组件之一，关键程度不言而喻。因此网络的问题也是社区放了很多力量在上面。下次的OpenStack版本是Okata,希望可以波士顿见～