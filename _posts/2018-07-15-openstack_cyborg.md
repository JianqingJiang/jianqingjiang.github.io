---
layout: post
title: Openstack Cyborg项目介绍
description: "Openstack Cyborg项目介绍"
tags: [OpenStack]
categories: [OpenStack]
---
###  Cyborg项目背景随着机器学习（machine learning）和机器视觉（compute vision）的快速发展，用户对GPU的需求也日益剧增。截止目前，大多数用户仍会选择带有GPU的裸机服务器。然而，这同时意味着用户需要承担由配置此类设备所带来的管理性成本。如今，用户将能够使用vGPU驱动的虚拟机，并利用这部分资源运行人工智能相关的Workload。  
随着OpenStack社区对AI和边缘计算的布局，而加速计算在边缘比在数据中心更为普遍，所以这又会加强OpenStack的地位，因此OpenStack在第17个版本迎来了Cyborg项目。  
Cyborg项目起源于NFV acceleration management以及ETSI NFV-IFA 004 document，和OPNFV DPACC项目。Cyborg（以前称为Nomad）是用于管理硬件和软件加速资源（如 GPU、FPGA、CryptoCards和DPDK / SPDK）的框架，在Queens发布中首次亮相。特别是对于有 NFV workload的运营商，计算加速已经成为云虚拟机的必备功能。通过Cyborg，运维者可以列出、识别和发现加速器，连接和分离加速器实例，安装和卸载驱动。它也可以单独使用或与Nova或Ironic结合使用。Cyborg可以通过Nova计算控制器或Ironic裸机控制器来配置和取消配置这些设备。  
在加速器方面，Nova计算控制器现在可以将Workload部署到Nvidia和Intel的虚拟化GPU（AMD GPU正在开发）。加速器可用于图形处理的场景（如虚拟桌面和工作站），还可以应用于集群上的通过虚拟化GPU以运行HPC或AI Workload的场景。  ### Cyborg组件架构
![1](/images/openstack-cyborg/1.png)   
Cyborg API---应该支持有关加速器的基本操作，API支持以下接口： 
 - attach：连接现有的物理加速器或创建新的虚拟加速器，然后分配给虚拟机  - detach：分离现有物理加速器或释放虚拟机的虚拟加速器  - list：列出所有附加的加速器  - update：修改加速器（状态或设备本身）  - admin：CRUD操作无关的某些配置  Cyborg Agent---Cyborg agent将存在于计算主机以及可能使用加速器的其他主机上，agent具体的作用： 
 - 检查硬件以找到加速器  - 管理安装驱动程序，依赖关系和卸载驱动  - 将实例连接到加速器  - 向Cyborg服务器报告有关可用加速器，状态和利用率的数据  -  硬件发现：  每隔数秒就会扫描实例的加速器和现有加速器的使用级别，并将这些信息通过心跳消息报告给Cyborg服务器，以帮助管理调度加速器  - 硬件管理：  Ansible将用于管理每个加速器的配置文件和加速器的Driver。install和uninstall特定的ansible playbook适配Cyborg所支持的硬件。在管理的硬件上进行的配置更改将通过运行不同配置的playbook作为底层实现  - 实例连接：  一旦产生一个实例需要连接到主机上的特定加速器，Cyborg服务器将向Cyborg agent发送消息。由于不同加速器之间的连接方法不同，因此agent需要不同的driver提供连接功能  Cyborg-Conductor---Cyborg-db的数据库查询更新操作都需要通过向Cyborg-conductor服务发送RPC请求来实现，conductor负责数据库的访问权限控制，避免直接访问数据库  openstack-Cyborg-generic-driver功能  
- 识别和发现附加的加速器后端- 列出在后端运行的服务- 将加速器附加到通用后端- 从通用后端分离加速器。- 列出附加到通用后端的加速器。- 修改附加到通用后端的加速器。Quata---cyborg resource quota，Cyborg的配额管理用于在构建虚拟机时管理用户或项目对加速器的访问。目前，项目或用户可能拥有无限数量的加速资源，应该有一个限制，限制是可配置的  ### Cyborg调用加速器过程（vFPGA）  
![2](/images/openstack-cyborg/2.png) 1.ronic监控网络并发现新资源  2.新的主机通过pXE启动并用Hypervisor初始化  3.Agent更新Nova和Neutron DB  4.Ironic agent根据存储在swift / glance / glare中的比特流加载静态区域  5.Nova agent被通知存在新的PCIe设备（来自SR-IOV的VF）并更新Nova DB  6.Nova根据用户指令需要孵化一台虚拟机并配备PR（vFPGA）  7.Nova过滤器找到可用资源并执行虚拟机创建/配置  8.VM cloud_init使用本地文件或Swift中的比特流加载PR---VM请求Cyborg从Glare加载PR  9.VF注册并分配给虚拟机  10.VM应用程序访问VF  总体来说，Cyborg的出现，在云主机中支持 vGPU（ 虚拟图形处理单元 ）的功能，这对于图形密集型工作负载以及许多科学性的、人工智能和机器学习的工作负载来说是一项重要的能力。  








