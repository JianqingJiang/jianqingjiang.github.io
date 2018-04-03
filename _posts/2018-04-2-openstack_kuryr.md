---
layout: post
title: Kuryr-一种更好的将容器网络与OpenStack网络融合的机制
description: "Kuryr-一种更好的将容器网络与OpenStack网络融合的机制"
tags: [OpenStack]
categories: [OpenStack]
---
###     Kuryr背景介绍

Kuryr项目在OpenStack big tent下的一个项目，目的是将容器网络与OpenStack Neutron对接。  

正式介绍前，先说下Kuryr这个单词。Kuryr是一个捷克语单词kurýr，对应英语里面是courier，对应的中文意思就是信使，送信的人。从这个名字能看出来，Kuryr不生产信息，只是网络世界的搬运工。
  
###     Kuryr解决了什么问题
OpenStack和Neutron已经是成立很久并且相对比较成熟的开源项目，特别是拿Neutron来说，Neutron现在有一个很完整的生态系统，并且有大量的plugin来为Neutron提供各种各样的网络解决方案，例如LBaaS、VPNaaS、FWaaS，并且这些plugin已经在逐步得被云用户在生产环境中应用。  
我们注意到容器网络方面，特别是在容器和OpenStack混合使用的环境中，每个网络解决方案都尝试为容器重新创建和启用网络，但这次使用Docker API（或任何其他抽象）OpenStack Magnum来对容器网络进行管理 
Kuryr背后的想法是能够利用Neutron及其插件和服务中的抽象和功能，来为容器用例提供生产级网络。 而不是使用每个独立的Neutron插件或解决方案来做，这样就引出了文章主题 - Kuryr。  
Kuryr的目标是成为两个社区Docker和Neutron之间的“集成桥梁”，进而推动Neutron（或Docker）项目的发展，以便能够满足容器网络的各种场景。  

重要的是要指出，Kuryr本身不是一个网络解决方案，也不是企图成为一个网络解决方案。 Kuryr的工作重点是成为为Docker提供Neutron网络和服务的信使。  

###     Kuryr架构
将Docker libnetwork映射到Neutron API  
下图显示了Kuryr架构的基本概念，将Docker和libnetwork网络模型映射到Neutron API  

![1](/images/kuryr/1.jpg) 

Kuryr映射libnetwork API并在Neutron中创建适当的对象，这意味着每个实现Neutron API的解决方案现在都可以用于容器联网。 Neutron提供的所有附加功能可以应用于容器端口，例如安全组，NAT服务和浮动IP  

但是，Kuryr的潜力并不止于核心API和基本扩展，Kuryr可以利用Neutron提供的网络服务及其特性，如LBaaS来实现Kubernetes服务的抽象等等  

当API的映射不是非常明显，并且如果需要驱动Neutron社区的变化，Kuryr也将缩小差距。 最近的例子是标签添加到Neutron资源，允许像Kuryr这样的API客户端存储映射数据和端口转发，以便能够提供端口暴露的Docker风格（当然，所有这些仍然在审查和批准过程中）  

###     提供通用的VIF绑定基础结构


Neutron解决方案需要支持容器网络的一个常见场景是，在这些场景中，缺乏新的端口绑定基础设施并且不支持libvirt。  

Kuryr试图为各种端口类型提供通用的VIF绑定机制，这些端口类型将从Docker命名空间端点接受数据，并根据其类型（或传递给它来完成绑定）将其附加到网络解决方案基础结构。  

对于运行在VM内部的容器运行的情况，VIF绑定也是需要的，这将在下一节中介绍。 这些VM不是由nova直接管理的，因此它们中没有任何OpenStack代理，这意味着需要一些机制来执行VIF绑定，并且可以从调用Shim Kuryr层的本地Docker远程驱动程序启动。 （但是这个问题仍然在讨论中）  

以下图表描述了使用Kuryr的VIF绑定  

![1](/images/kuryr/2.jpg) 

提供Neutron插件和Kolla集成的Containerized image  

Kuryr的目标是提供各种Neutron插件的一个集成工具箱，并且有望与kolla集成。  
这意味着我们将为各种Neutron插件提供各种image，例如具有Kuryr层的OVS L2 Agent，Midonet，OVN。 Kolla集成将带来运营商所需的易用部署，同时不会失去对服务间认证和可配置性的控制。  

###     嵌套的VM和Magnum用例

Kuryr希望解决的另一个用例是在用户拥有的OpenStack创建的VM（即VM容器）上部署容器的常见模式。 这种部署模式通过将用户拥有的容器与运营商拥有的Nova Compute机器分离，为容器部署提供了额外的安全层。  
这个用例也被OpenStack Magnum使用，Magnum是由OpenStack Containers Team开发的OpenStack API服务，使得Docker和Kubernetes等容器编排引擎成为OpenStack中的一流资源。  

已经有Neutron插件支持这种用例，并且提供了一种优雅的方式来将嵌套容器连接到不同于VM本身的网络的不同逻辑网络中，并在其上应用Neutron特性，例如OVN  


使用Kuryr，OVN和MidoNet等其他Neutron供应商可以利用通用基础设施与Docker进行交互，并将重点放在完善Neutron API，并推动网络向前发展  

###     Kuryr工作机制
一、docker那边需要先挖好坑，把网络部分的接口暴露出来。  

* docker把网络部分提取成了libnetwork，并为此实现了几个driver，host、overlay、null和remote，其中remote就是为用其他项目来管理docker的网络留的接口。remote采用rpc的方式将参数通过网络发给外部管理程序，外部程序处理好了后通过json返回结果给remote。
* libnetwork的remote driver定义了基本的创建网络、创建port的接口，只要对应的REST Server实现了这些接口，就可以提供整套docker的网络。这些接口有：Create network、Delete network、Create endpoint、Delete endpoint、Join（绑定）、Leave（去绑定）。
* remote driver的接口定义在 https://github.com/docker/libnetwork/blob/master/docs/remote.md
* 用remote driver分配了设备一般是不带IP的，libnetwork使用ipam driver来管理ip。这些接口有：GetDefaultAddressSpaces 获取保留IP，RequestPool获取IP地址池，ReleasePool释放IP地址池，RequestAddress获取IP地址，ReleaseAddress释放IP地址。
* ipam driver的接口定义在 https://github.com/docker/libnetwork/blob/master/docs/ipam.md *  libnetwork的插件发现机制在 https://github.com/docker/docker/blob/master/docs/extend/plugin_api.md#plugin-discovery

二、kuryr这边就按照docker给的接口转为neutron的操作。  
  
* 配置文件里面定义好参数，以便能连上neutron。
* 使用Flask（而不是wsgi）实现一个REST Server，以便接收remote driver传过来的参数。
* 使用libnetwork的插件机制，在 /usr/lib/docker/plugins/ 目录下建立kuryr的插件描述的json文件。
* docker daemon启动的时候，libnetwork发现kuryr这个新插件，并且通过 /Plugin.Active 验证这个插件可用。
* 使用docker network create --driver=kuryr foo 在docker中创建一个网络
* 用户使用docker创建一个容器的时候，指定网络为 foo
* kuryr从libnetwork接收到请求，并用很多硬编码的schemata来验证用户输入的参数对不对，然后把创建 Network/Sandbox/Endpoint转为对应neutron的资源，发送给neutron。
* kuryr接收neutron的结果，再用pyroute2创建veth pair，一个bind到neutron的port，一个以给veth设置netns的方式给容器。
* kuryr把neutron返回的结果包装成libnetwork的样式，转发给libnetwork。
这样容器就能用neutron的网络了，并且可以使用因此而来的附加功能：安全组、租户网络。。。。
* kuryr类似ovs-agent，也需要每个计算节点安装一个。




