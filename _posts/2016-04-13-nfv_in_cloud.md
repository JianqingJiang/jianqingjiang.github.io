---
layout: post
title: 虚拟实验室NFV实现及测试方法研究
description: "虚拟实验室NFV实现及测试方法研究"
tags: [NFV]
categories: [NFV]
---
### 写于2016年，作为本科毕业论文

### 约定：1. 虚拟NFV网元＝Virtual Router(虚拟路由器)

###  2. 英文标题 How can Virtual Router live in the Cloud

##    摘要
IT耗资在前所未有的剧增。企业和运营商在使用云计算技术来降低成本的同时自然而然的想到使用低成本的x86服务器作为硬件，在这之上使用虚拟的路由器网元来节省成本，来代替路由器厂商的高价路由器。  

##    目标与内容

目前大多数的教育机构的实验室实体设备都存在灵活性低的问题。虚拟实验室一方面不断在专用硬件设备、基础设施上加大投资，另一方面学生在实验拓扑变得越来越复杂，占用的物理设备的数量增加。这就是所谓的“剪刀差”。当设备类型越丰富、数量越多、拓扑越多样的时候，用户能够享受的教学规模也越大。但由于资金、场地等各方面的因素限制，实体实验室能够提供的设备数量有限，设备的类型也单一，一般都是指定某个厂商、固定的设备配置等。可支持学生进行网络实验的网络拓扑也无法多样化。基本上大中型实验项目无法拓展，因此也导致用户无法将自身所学的专业技能发挥到实际项目中，网络规划与设计综合技能有待提高。  

##    项目需求

###    需求分析
仿真软件不能进行“网络测试分析”实验课，也不方便实验室的统一管理，因此需要引进NFV虚拟网元；实验室网线部署和设备的部署增加了运维；物理设备价格昂贵且容易宕机需要部署虚拟环境云计算OpenStack平台；云计算平台与NFV虚拟网元的结合才能搭建完整的虚拟实验平台提供给师生使用。NFV虚拟网元之间需要有虚拟化支持和网络连通性，才能进行组网完成网络相关实验。使用NFV虚拟的测试网元对整个环境进行测试，保证虚拟实验平台的可用。  
而且由于物理设备的有限。学生做实验的时候通常需要分批来进行，要是需要某一台物理的设备宕机了，可能就导致实验进行不下去。因此在物理设备上做实验的问题更加容易出现。一旦物理设备的宕机，如果采购不及时，那么便会长时间影响做实验的效率和课程的进度。  
![虚拟实验室需求](/images/nfv_in_openstack/1.png)

### 特定NFV虚拟网元的需求
在OpenStack平台中部署的NFV虚拟网元需要满足“路由与交换”以及“网络性能测试”应该具备以下的特点。  






##   技术背景
虚拟实验室部署的OpenStack需要满足物理资源虚拟化的基本需求。而且由于是生产环境，必须提供高可用部署方案。对NFV虚拟网元的选择上，需要满足路由与交换的配置命令，本文选择了VyOS和OpenWRT。在OpenStack虚拟化技术中Nova调用Libvirt技术。而OpenStack网络技术中Neutron是由虚拟的接口，网桥和Open vSwitch组成的，Neutron提供ML2 Plugin的启动网元方式。在测试中，利用了Spirent的STC软件，vSTC虚拟端口以及Velocity界面。  

![虚拟实验室OpenStack组件介绍](/images/nfv_in_openstack/2.png)  
在虚拟实验室环境中，为了兼顾性能和可靠性，Keystone进行访问管理，Nova组件进行虚拟化的支持技术，Glance进行镜像的存放和管理，Neutron则是对OpenStack的网络进行管理和调度。Horizon是提供Web界面可以使得温州大学的师生能够访问。最后使用Cinder和Swift进行存储资源的管理调度。  


![Nova组件结构](/images/nfv_in_openstack/3.png)  
Libvirt为各种虚拟化工具提供一套方便、可靠的编程接口，用一种单一的方式管理多种不同的虚拟化提供方式。  
![Libvirt支持的虚拟化平台](/images/nfv_in_openstack/4.png)  
Nova调用底层虚拟机监控程序（如KVM/ QEMU等）和对虚拟化功能的管理主要通过LibvirtDriver类来实现。  
![网络设备的虚拟化形式](/images/nfv_in_openstack/5.png)  
1）TAP/TUN/VETH  
![Neutron组件结构](/images/nfv_in_openstack/6.png)  
2.Neutron Plugin  
ML2实现的网络/子网/端口三大核心资源，ML2实现网络拓扑结构（扁平，VLAN VXLAN，GRE）和底层虚拟网络（Linux的网桥，OVS）分离机构的类型，并使用扩展驱动的方式。其中，Mechanism Manage管理不同种类的的驱动程序，不同的网络实施机制对应不同的网络形式（如Linux bridge，OVS，NSX等）。  





OpenStack环境中的Compute节点的Libvirt配置文件中可以对KVM，QUME等虚拟化技术进行选择。在裸机上的部署情况下一般使用KVM技术，因为它使用硬件加速，可以让计算资源分配时间更加短。虚拟实验室在进行搭建的时候使用了KVM虚拟化技术，而QEMU则是软件实现的虚拟化，没有使用硬件加速，使得NFV虚拟网元部署的时间变长，它的优点则是平台的兼容性较强。  
Libvirt中存在libvirtd进程，libvirt中有Qemu和KVM的驱动。在本项目中选择性能较好的KVM作为虚拟化层，在KVM层之上就是虚拟化所生成的Domain，Domain为NFV虚拟网元提供了计算资源。  
1.NFV虚拟网元管理：针对NFV虚拟网元进行生命周期操作。针对具体NFV虚拟网元上则是孵化，关机，快照，删除等功能。  


而加入的NFV虚拟网元（开源版本）不能作为一个插件，而VyOS的Brocade商业版本Vyatta在OpenStack中官方提供了插件，可以用来替换OpenStack中原本存在的L3-agent。关于VyOS的Brocade的Plugin。直接是Create Router而不是启动节点。但是在OpenStack的Controller节点上并不能安装Plugin，Brocade的Plugin是不支持开源VyOS。作为Plugin形式的启动是可行的，需要对每一个特定的NFV虚拟网元都进行开发一个特定的Plugin，和OpenStack的网络进行通信。OpenStack提供了Plugin的热插拔设计方式，使得这一切变得可以实现。那么这个方式的优势是利用OpenStack本身的框架，可行性较强。缺点也很明显，NFV的虚拟网元具有独特性，需要对每一个虚拟网元单独开发一个特定的Plugin，开发工作和难度都较大。 
 
OpenStack的安装方式是在物理机上安装Ubuntu再安装OpenStack，版本是Juno，需要物理磁盘的分区及安装。VyOS在OpenStack中直接启动会有一个安装的过程与OpenStack不兼容，在启动VyOS的时候出现了：sda dirves not found的问题。需要在Virtualbox或者VMware上进行install system。而OpenWRT则不需要这个过程。然后在Linux发行版上安装KVM。进行虚拟格式的转换。可以转换成qcow2或者raw格式等。  



方法spawn：
```

为新建立的NFV虚拟网元参数获取配置数据conf，并把获取的数据conf转换为xml格式；  
![XML配置信息](/images/nfv_in_openstack/14.png)  
这样可以得到XML配置文件/var/lib/nova/instances/instance_id/libvirt.xml  

```
```


![网络连通性分析图](/images/nfv_in_openstack/15.png)  
在 Horizon上创建Neutron网络过程如下：首先管理员拿到一组公有IP地址，并且创建一个外部网络和子网。租户创建一个网络和子网。租户创建一个路由器并且连接租户子网到外部网络。  
![虚拟网元流量分析图](/images/nfv_in_openstack/16.png)  
图中NFV虚拟网元流量从vm01到vm02需要通过eth0，到vnet0，接着通过qbr(Linux bridge)，再经过一对虚拟接口qvb以及qvo。然后到达OpenvSwitch，从br-int经过int-br-eth，接着经过物理网络之后就向上经过同样的虚拟设备最后到达vm02。由图中可见，流量在经过eth1时进行了vlan的tag转换，因此vlan不是影响跨网段Instance连通性的障碍。  
![OpenvSwitch结构图](/images/nfv_in_openstack/17.png)  

###   NFV 虚拟网元的Namespace
在这里也是采用这样的原理，在Neutron中我们可以使用命令将两个需要相互连通的网络加入到Router Network NameSpace中，这时Router Network NameSpace中就会出现两个网络接口连接到OVS bridge-int上，并且这两个网络接口的IP地址就是两个网络的网关。除此以外，我们需要在Router Network NameSpace中设置net.ipv4.ip_forward参数为1，这样就可以在Router Network NameSpace内部进行不同网段数据的转发，至此两个不同网络就可以打通了。  



![OpenStack L3-agent NameSpace连接方式](/images/nfv_in_openstack/18.png)  
如果NFV的虚拟网元（VyOS、OpenWRT）作为Instance启动，那么OpenStack后台是这样的命令，创建虚拟网元的Namespace然后去连接qrouter的Namespace。  

```


![OpenStack NFV虚拟网元 NameSpace连接方式](/images/nfv_in_openstack/19.png)  
那么Instance的Namespace和OpenStack L3-agent所创建的Router拥有不一样的Namespace的标示符和连接方式。Openstack不认为作为Instance启动的VyOS给分配一个Router类型的NameSpace。因此作为Instance启动的NFV虚拟路由器和OpenStack本身的路由器的不同点主要是qrouter的Namespace是使用“qr”虚拟网络组件进行连接的，而作为Instance启动的NFV虚拟路由器的Namespace是通过Linux bridge的“qbr”进行连接的。那么总结起来，不同点就是Namespace的连接方式上，即Linux bridge的连接导致的网络通信中断。  

![L3-agent NameSpace网络监测](/images/nfv_in_openstack/20.png)  
Debug流量在哪里被Drop  
![NFV虚拟网元NameSpace网络连接监测](/images/nfv_in_openstack/21.png)  
在实验过程中qbr上可以监测到OSPF的hello包，由于NFV虚拟路由器中已经配置了OSPF的路由协议，因此会广播hello包。由于在实验连接vm02中qbr上收到了来自vm01的OSPF的hello。那么可以得出结论。包已经经过了连接vm01的虚拟接口，Linuxbridge形式的qbr以及经过了OpenvSwitch，从而到达了连接vm02的Linuxbridge形式的qbr。  

![实例的链和规则](/images/nfv_in_openstack/22.png)  
在OpenStack的实例对应iptables所创建的链和规则来看有以下几个特点：  
![iptables的应用](/images/nfv_in_openstack/23.png)  
那么对于连接多个OpenStack子网的Instance（NFV虚拟网元）是对应第4点，所以所有到另一个OpenStack子网的Instance（NFV虚拟网元）的全部流量被iptables所Drop。具体表现就是两个虚拟网元之间不能进行通信。  
![特定NFV虚拟网元的iptables的定位](/images/nfv_in_openstack/24.png)  

```
!/usr/local/bin/env python
```



###   小结


在Velocity中可以调用在OpenStack的Glance中的Spirent TestCenter Virtual（vSTC）,通过vSTC与NFV虚拟网元的联合组网，可以满足网络工程的“路由与交换”实验以及“网络性能测试”等实验课的需求。Velocity作为温州大学虚拟实验室的WEB接口，用户通过用户名和密码登录。  
![学生Web界面](/images/nfv_in_openstack/25.png)  

### Spirent TestCenter测试实现
![Spirent TestCenter的界面](/images/nfv_in_openstack/26.png)  
测试是基于RFC2544协议的。RFC2544协议是RFC组织提出的用于评测网络互联设备的国际标准。吞吐量测试是被测设备在不丢包的情况下，所能转发的最大数据流量。用户以一个用户定义的恒定速度发送，然后通过二分查找算法找到一个不丢包的速率。结果是在不同的帧长下每秒的吞吐量。常见的帧长有 64，512，1024,1518字节等。流量通过VyOS的测试结果如图  
![VyOS吞吐量测试报告](/images/nfv_in_openstack/27.png)  
这样整个虚拟实验室的NFV虚拟网络的功能和性能已经满足了上述的需求。  



![虚拟实验室整体架构](/images/nfv_in_openstack/28.png)  

## 参考文献