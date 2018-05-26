---
layout: post
title: OpenStack Zun组件详解
description: "OpenStack Zun组件详解"
tags: [OpenStack]
categories: [OpenStack]
---

###   什么是ZUN？
Zun是Openstack中提供容器管理服务的组件，于2016年6月建立。Zun的目标是提供统一的Openstack API用于启动和管理容器，支持多种容器技术。Zun原来称为Higgins，后改名为Zun。Zun计划支持多种容器技术，Docker，Rkt，clear container等，目前只支持Docker。  OpenStack Queens版本发布，由于容器社区的火热，一项值得关注的补充则为“Zun”，它在OpenStack项目中负责提供容器服务，旨在通过与Neutron、Cinder、Keystone以及其它核心OpenStack服务相集成以实现容器的快速普及。通过这种方式，OpenStack的原有网络、存储以及身份验证工具将全部适用于容器体系，从而确保容器能够满足安全与合规性要求。  ###   Zun集成了OpenStack什么服务？Zun需要下列的OpenStack的服务来支持：  
* Keystone* Neutron* Kuryr-libnetworkZun也可以集成下面的OpenStack的服务（可选）：  
* Cinder* Heat* Glance在使用Zun的时候，可以直接调用Zun的自带工具或API来创建和管理Docker的Workflow。 Zun的用户功能（以及某些管理员功能）都通过REST API公开，可以直接使用。  
另外，也可以通过其他OpenStack组件的API或者SDK来间接调用Zun的API
 * Horizon: 通过OpenStack WebUI来调用* OpenStack Client: 通过OpenStack CLI来调用* Zun Client: 通过Zun的Python client来调用###   Magnum与ZUN的区别Magnum是OpenStack中一个提供容器集群部署的服务，通过Heat部署虚拟机和物理机，组成集群，然后调用COE接口完成容器的部署。Magnum项目创建之初，项目目标以CaaS为宗旨，即容器即服务；在后续的发展中将功能集中在容器的集群部署上。  Zun和Magnum的差异在于Zun目标是提供管理容器的API，而Magnum提供部署和管理容器编排引擎（COE）的API。Zun不准备实现COE提供的很多先进的功能（例如容器保活、负载均衡等），而是提供基本的容器操作（CRUD），并和Openstack紧密集成。  ###   ZUN架构
为了更好理解zun与OpenStack其他组件的关系，下面是zun的架构图  
* Zun API: 处理 REST请求并确认输入参数* Zun Compute: 启动容器并调度计算资源* Keystone: 认证系统* Neutron: 提供容器网络* Glance: 用于存储docker镜像（另一种选择是使用DockerHub）* Kuryr: 用于连接容器网络和OpenStack Neutron的一种plugin ![1](/images/zun/1.jpg) ###   ZUN 怎么使用？Zun主要使用Container和Capsule的概念。除了Container概念之外，Zun还提供了一些稍微先进的抽象概念，如Capsule。 Zun Capsule的概念有点像Kubernetes Pod，代表一组容器。 Capsule用于将多个需要彼此紧密合作的容器分组，以实现业务目标。  Container由Docker或其他容器引擎支持。Zun集成了基本的Docker的功能（如创建/删除容器）。   Zun的操作要求与其他OpenStack服务相似。 它需要在控制器节点上安装Zun API server，并在每个计算节点上安装Zun-agent。 Zun的主要依赖是一组OpenStack Base Services：与oslo兼容的数据库，消息队列，Etcd和Keystone。  此外，Zun还为容器添加了多个由OpenStack提供的功能。 例如，默认情况下，Zun容器可以使用Neutron分配的IP地址，可以attach Cinder卷，使用由Keystone提供的认证服务。 对于OpenStack用户，他们会发现开始使用Zun容器相当容易。  将Zun与Neutron一起使用，可以在Nova实例所在的隔离网络环境中创建容器。 VM的Neutron功能（即安全组，QoS）也可用于Zun容器。  下面命令就是利用了Neutron和Kuryr来为zun容器提供网络服务的： 
 ```
$ openstack appcontainer run --net network=net1 \    --net network=net2,v4-fixed-ip=10.0.0.31 \    --net port=net3-port \nginx:latest```为了容纳需要保存数据的应用程序，常用的方法是利用外部服务为容器提供持久卷。 Zun通过与OpenStack Cinder集成解决了这个问题。 创建容器时，用户可以选择将Cinder卷装入容器。 Cinder卷可以是租户中的现有卷或新创建的卷。 每个卷将被绑定到容器文件系统中的路径中，并且存储在那里的数据将被持久化。下面命令就是利用Cinder来持久化存储容器数据的：  
```$ openstack appcontainer run \    --mount source=my_cinder_volume,destination=/path/in/container \nginx:latest```在Orchestration方面，与其他提供内置编排的容器平台不同，Zun使用外部编排系统来实现此目的（现在，Zun与OpenStack Heat集成，将来，Zun将与Kubernetes集成）。通过使用外部协调工具，最终用户可以使用该工具提供的DSL定义他们的容器化应用程序。特别是，借助Heat，还可以定义由容器资源和OpenStack资源组成的资源，例如Neutron负载平衡器，浮动IP，Nova实例等。  ###   ZUN和Kubernetes是竞争关系吗？Zun和Kubernetes是互补的关系，事实上，Zun社区正在积极地在与Kubenetes集成。目前Zun与COE集成的工作主要还是在Kubenetes上，与其他COE集成还需要时间。  Kubernetes使容器更易于部署，管理和扩展。 但是，在OpenStack上使用Kubernetes仍然需要用户手动底层基础设施，如虚拟服务器集群。 用户需要负责初始容量规划，例如决定VM群集大小以及正在运行的VM群集的维护。  Serverless容器技术或解决方案（如亚马逊网络服务（AWS）Fargate，Azure容器实例（ACI）和OpenStack Zun）的出现为在云上运行容器提供了一种可行的替代方案。 Serverless方法允许用户按需运行容器，而无需预先创建或管理自己的集群。  Zun将利用Kubernetes作为编排层，Kubernetes使用OpenStack Zun来提供“Serverless”容器。   ###   ZUN和Kubernetes结合实现细节：Virtual-kubelet这个项目可以完成kubelet 到Zun的转换，需要在virtual-kubelet这个项目里面注册一下Zun的driver  感兴趣的同学可以点击下面链接：  
```https://github.com/virtual-kubelet/virtual-kubelet```
如果您选择私有云解决方案，则可以使用OpenStack Zun构建无服务器容器云。 但是，如果需求是独立的基础架构的解决方案，则可以选择使用平台级别的工具，例如Kubernetes。将来，可以将Kubernetes连接到无服务器技术，以便可以跳过配置Kubernetes节点集群并按需启动Kubernetes Pod的步骤。  ###   ZUN-UI 预览创建 Container 的页面和创建虚拟机的页面类似。可以选择来自 DockerHub 或者 Glance 的镜像，可以选择网络、端口、安全组等，支持容器特有的一些功能如限制cpu和内存的使用  
![2](/images/zun/2.jpg)  对容器的一些基本操作：更新、停止、重启、暂停、执行命令、删除等操作   ![3](/images/zun/3.jpg) ###   总结Zun 的操作基本和 docker 的操作一致，使用起来和原生docker容器没有区别。但是现在可以和 Openstack 的资源良好的结合在一起，统一管理，提高的 OpenStack容器管理的灵活度，还是很令人期待的。  ###   参考资料https://docs.openstack.org/zun/latest/  Wiki: https://wiki.openstack.org/wiki/Zun  
https://www.linkedin.com/pulse/aws-fargate-openstack-zun-comparing-serverless-container-hongbin-lu/?from=timeline&isappinstalled=0  Source code: https://github.com/openstack/zun  


