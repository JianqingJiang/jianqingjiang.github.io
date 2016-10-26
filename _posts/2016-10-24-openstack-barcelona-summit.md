---
layout: post
title: OpenStack巴塞罗那峰会Day2
description: "OpenStack巴塞罗那峰会Day2"
tags: [OpenStack]
categories: [OpenStack]
---


##  简述

第二天的峰会先是在大会场中进行Keynote的介绍。中国移动的出场率比较高。作为世界上最大的移动电话的服务商，已经率先使用OpenStack了。混合云正在成为今后企业云形态的主流，在Best practice的session里面大会也在强调企业正在不断得推进OpenStack,容器和k8s也占有会议的一席之地，在越来越多的企业加入进来之后，安全问题也被提上议程。 

![1](/images/openstack-barcelona-summit-2/1.jpg)  

## KEYNOTE 

##  1.The future is Multi-Cloud  
What is mutil-cloud look like?  
大会花了很长的时间来介绍他们的CI/CD system  
World's lagest mutil-cloud is CI/CD system  
1.7 million test jobs every day  
10-20k vms per day  
gerrit/zuul/nodepool  
2000+ jobs per hour  
5000 PB every two day  
社区正在帮助更多的企业参与到OpenStack中。其中一个Demo是把OpenStack的虚拟机传到AWS上，这个Demo也是很符合今天Multi-Cloud的主旨  

##  2.Best Practice Share
IBM也在向云转型，投入了很大的精力  
What proving "interop" means for the ecosystem?    
Materna acts a s service provider  
Materna acts as software developer  
Materna acts as trusted advisor/platform designer  

Why head for OpenStack?  
Scalability  
Speed  
Cost  
Interoperability  

platform9 system做了一个关于为什么他们的商务伙伴选择OpenStack的调查。OpenStack的标准化的API这个优势是最多企业选择的 

![2](/images/openstack-barcelona-summit-2/2.jpg) 


## 3.Containers in bare metal

容器技术的重要在本次峰会中又被再一次的验证了  
嘉宾分享关于容器的技术

Challenges  
KVM  
Cost  
Bootstrapping  
Small Team  

Advantage  
bare Metal  
perfomance  
cost effective  
fast deployment  

Containers  
give control back to developers  
simple bootstrapping  
safe deployments  
mix match stacks  
 


在昨天很多关于容器的分享中，今天的keynote又提到了很多容器技术，Kolla,Magnum作为一个社区的明星项目，也是社区正在发力的地方，在分享中k8s的使用频率比其他的mesos等容器管理技术高。  

![3](/images/openstack-barcelona-summit-2/3.jpg)  

社区正在攻克Magnum的网络层面。即使用Neutron网络对容器的网络进行管理。社区提到了容器的不同部署方式的管理，即针对Docker的bare metal和容器跑在虚拟机中这两种部署方式，Magnum都会一视同仁得对容器进行管理，以及对网络层面进行不同方式的连接。  

Magnum的小组讨论也是最激烈的  

![8](/images/openstack-barcelona-summit-2/8.jpg)


##  4.Hackthon
Hackthon也是社区正在力推的一个活动，Hackthon帮助更多的人参与到社区里面，对项目提出意见贡献代码等。也是因为更多的人拥有开发OpenStack的能力，所以社区推出的OpenStack的认证也是水到渠成。  


Design Summit是今后峰会的亮点，也正式与之前的Hackthon和OpenStack认证一脉相承，社区正在鼓励大家对OpenStack出谋划策。  

![4](/images/openstack-barcelona-summit-2/4.jpg)  


##  分会场：一些有趣的特性

网络层面富士公司对于DVR网络的防火墙问题，另外对于Neutron的网络难于troubleshooting的问题做了“neutron logging”地项目  

![5](/images/openstack-barcelona-summit-2/5.jpg)  
![6](/images/openstack-barcelona-summit-2/6.jpg)  
![7](/images/openstack-barcelona-summit-2/7.jpg)  

OpenStack社区也在积极得向Opendaylight社区以及OPNFV社区合作，也符合OpenStack的“big tent”的一贯作风  

![9](/images/openstack-barcelona-summit-2/9.jpg)



