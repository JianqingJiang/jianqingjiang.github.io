---
layout: post
title: Neutron中Linux bridge与Open vSwitch两种plugin优劣势对比
description: "Neutron中Linux bridge与Open vSwitch两种plugin优劣势对比"
tags: [OpenStack]
categories: [OpenStack]
---

目前说到虚拟交换机，通常会想到使用Open vSwitch做虚拟交换机，因为支持Open vSwitch的个人和企业都想要有一个开放的模式把他们的服务集成进OpenStack。 Open vSwitch社区做了大量的工作，希望提升Open vSwitch作为最主要虚拟交换机的地位。社区期望Open vSwitch将在软件定义网络（SDN）接管网络时代到来时，提供所有可能最好的交换服务。但是，Open vSwitch的复杂性使用户渴望更简单的网络解决方案，在这过程中需要Linux Bridge这样的简单桥接技术来支持云解决方案。  

但是Open vSwitchh支持者会指出Linux Bridge 缺少可扩展性的隧道模型。Linuxbridge是二层概念，不能支持GRE的三层隧道技术，vxlan模型是在linux kernel较高版本（3.9以上）才native支持的。因此有这些观点的网络专家们，他们会比较坚定的认为复杂的解决方案比一个简单的解决方案要好。  

当然，Linux Bridge已经发生了变化，也有助于缩小使用Open vSwitch和Linux Bridge的之间的差距，包括添加VXLAN支持隧道技术。但是在更大的网络规模中，Linux Bridge的简单性可能会产生更大的价值。  

我们都知道，OpenStack 社区官方的安装文档的步骤在liberty版本之前都是以Open vSwitch为例子的。而且从OpenStack 用户调查来看，使用 Open vSwitch的人比使用 linux bridge 多很多。
  
Liberty 版本之前社区官方文档都是使用 neutron-plugin-openvswitch-agent， 但是 Liberty 版本转为使用 neutron-plugin-linuxbridge-agent。社区文档只留了这么一句话，意思是Linuxbridge更加简单。  

“In comparison to provider networks with Open vSwitch (OVS), this scenario relies completely on native Linux networking services which makes it the simplest of all scenarios in this guide.”  

以下是Open vSwitch与Linux bridge之间的优劣势对比  

（1）Open vSwitch 目前还存在不少稳定性问题，比如：    
* Kernetl panics 1.10
* ovs-switched segfaults 1.11
* 广播风暴
* Data corruption 2.01  

（2）于Linux bridge，Open vSwitch有以下好处：  
* Qos配置，可以为每台vm配置不同的速度和带宽
* 流量监控
* 数据包分析
* 将openflow引入到ovs中，实现控制逻辑和物理交换网络分离  

（3）为什么可以使用 Linux bridge？  
* 稳定性和可靠性要求：Linux bridge 有十几年的使用历史，非常成熟。
* 易于问题诊断 （troubleshooting）
* 社区也支持
* 还是可以使用 Overlay 网络 VxLAN 9需要 Linux 内核 3.9 版本或者以上)  

（4）使用 Linux bridge 的局限性  
* Neutron DVR 还不支持 Linux bridge
* 不支持 GRE
* 一些 OVS 提供但是 Neutorn 不支持的功能  

（5）长期来看，随着稳定性的进一步提高，Open vSwitch 会在生产环境中成为主流。  

![1](/images/ovs_compare_lb/1.jpg)   

Linux bridge 和 Open vSwitch 的功能对比：  

可以看出：  
（1）OVS 将各种功能都原生地实现在其中，这在起始阶段不可避免地带来潜在的稳定性和可调试性问题  
（2）Linux bridge 依赖各种其他模块来实现各种功能，而这些模块的发布时间往往都已经很长，稳定性较高  
（3）两者在核心功能上没有什么差距，只是在集中管控和性能优化这一块Open vSwitch有一些新的功能或者优化。但是，从测试结果看，两者的性能没有明显差异：  

![1](/images/ovs_compare_lb/2.jpg)   

![1](/images/ovs_compare_lb/3.jpg) 

总之，目前，Open vSwitch与Linux bridge都有各自的适合的场景，对于云用户来说也提供了更好的两种优秀的网络解决方案，除了SDN对集中管控的需求，和更新更多的网络特性时，Open vSwitch更加有优势，但是在稳定性，大规模网络部署等场景中Linux bridge 是个较好的选择。  



