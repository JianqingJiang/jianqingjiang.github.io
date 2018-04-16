---
layout: post
title: SDN网络浅析与选型
description: "SDN网络浅析与选型"
tags: [SDN]
categories: [SDN]
---
####   SDN概念  SDN的概念已经流行了很多年了，从一开始的实验室产品到2012年谷歌宣布其主干网络已经全面运行在OpenFlow上，使广域线路的利用率从30%提升到接近饱和。从而证明了OpenFlow不再仅仅是停留在学术界的一个研究模型，而是已经完全具备了可以在产品环境中应用的技术成熟度。  SDN社区力量在2014年的时候还很薄弱，控制器才只有floodlight、Ryu和极不成熟的opendaylight，科研机构和高校都还在用mininet在做模拟实验，后来有了运营商主导的ONOS控制器、Juniper OpenContrail和华为的DragonFlow，和较为成熟的OpenvSwitch情况才有所好转，可以利用X86服务器搭建小型的SDN网络。  到现在出现了很多白牌交换机厂商，有了很多商业的优秀的SDN控制器，和不仅在数据中心的场景，还有了SD-WAN等广域网场景，而现在SDN的概念也在扩大，随着OpenStack云计算的火热，由Neutron组件等优秀开源的项目，和集成的大量功能的Plugin也在推翻传统的SDN的定义。随着用户对SDN的追捧，和云计算项目的落地，不得不说SDN现在已经在大量的用户的生产环境中使用。  
#### SDN网络选型SDN网络的优势在这里就不展开讨论了，SDN目前有很多组网方案，来满足不同的用户场景。  ##### (一)控制器通过南向协议来控制白牌交换机的转发  第一种实现，就是纯南向协议的实现不支持任何传统协议。通过SDN控制器来发现全网拓扑，计算路径，再通过南向协议下发流表，实现流量按策略转发。如下图所示。
![1](/images/sdn-network/1.jpg)   在云计算的场景中，SDN控制器可以利用OpenStack Neutron来实现控制平面，通过OpenFlow/Netconf控制 VXLAN VTEP、GW、IP GW。它的优点是可以通过软交换机来实现网络的创建、子网的划分、路由的选择以及防火墙策略的管理等功能。##### (二)松散控制模式方案（通过网络设备控制协议自学习）  VxLAN隧道自动建立以及隧道自动关联；通过标准EVPN完成隧道建立和地址学习，利用EVPN的BGP RR实现邻居发现 ，每个设备都通告自己的VxLAN信息，每个VTEP设备都有全网的VXLAN信息以及VxLAN和下一跳的关系。VTEP设备会和那些跟自己有相同VXLAN的下一跳自动建立VXLAN隧道，并将此VxLAN隧道跟这些相同的VxLAN关联。SDN控制器只负责下发服务策略，不下发控制流表，可靠性更高。注意这里是VxLAN是属于在Underlay进行网络传输的。  MAC/IP route通过MP-BGP传输到对端VTEP。现实中要求BGP连接是full mesh(任意两两互连)，而为了减轻配置压力，通常会引入BGP RR(Router Reflector)。BGP RR的作用是将一个BGP Speaker的数据反射给所有其他连接的BGP peer。使用BGP RR可以使得所有的BGP Speaker只需与BGP RR建立连接，否则按照full mesh，任意一个BGP Speaker必须与其他所有的BGP peer建立BGP连接。  
![2](/images/sdn-network/2.jpg)   ##### （三）VLAN模型组网  在OpenStack的环境中Neutron网络组件可以选择VLAN组网模型。VLAN模型的特点缺点是需要在传统的网络交换机上放开对应的VLAN，因此在做网络接入时需要依赖网络的规划，同时在多租户隔离上不是特别方便，仍然需要在网络交换机上做操作，即网络组网不够灵活；优点是VLAN技术比较成熟，不管是性能上还是稳定性上都比较好。  
####  尾语在数据中心，大致有三种方式部署软件定义的网络。一种方法是使用OpenFlow/Netconf，一种SDN的南向标准，但是经常因可扩展性差而被诟病。一种更受欢迎的方式是使用虚拟网络覆盖，即使用VxLAN Overlay的方式扩展组网。这种技术是目前应用最广泛的。另外一种主流的技术是根据开放的协议，称为BGP EVPN，这是一种在数据中心将分别属于不同的租户之间的虚拟机进行流量隔离。其非常强大，是因为其能够实现让分别位于数据中心不同部分或完全位于不同的数据中心的虚拟机之间的专用连接。其也能够使整个虚拟机从一台主机设备迁移到另一台。