---
layout: post
title: OpenStack cinder mutil-attach技术探秘
description: "OpenStack cinder mutil-attach技术探秘"
tags: [OpenStack]
categories: [OpenStack]
---###  背景OpenStack 作为开放的 Infrastracture as a Service云操作系统，支持业界各种优秀的技术，这些技术可能是开源免费的，也可能是商业收费的。 这种开放的架构使得 OpenStack 保持技术上的先进性，具有很强的竞争力，同时又不会造成厂商锁定（Lock-in）。 那 OpenStack 的这种开放性体现在哪里呢？一个重要的方面就是采用基于 Driver 的框架。  
Cinder组件，存储节点支持多种 volume provider，包括 LVM, NFS, Ceph, GlusterFS，以及EMC, IBM等商业存储系统。cinder-volume为这些volume provider定义了统一的driver接口，volume provider只需要实现这些接口，就可以driver的形式即插即用到 OpenStack 中。  
目前，Cinder只允许将卷连接到单个主机或实例。但是有时用户可能希望能够将相同的卷附加到多个实例。在多年的上游社区的努力后，终于在Queens版本发布了这个期待已久的功能。  
### 案例需求能够使同一块卷提供给不同的用户使用，卷本身可以是read-only的也可以是read/write。  
###场景：![1](/images/cinder_mutilattach/1.png)云平台中的应用可能由多个虚拟机组成集群的方式提供，其中一台是active，一台是passive状态，若通过cinder mutil-attach方式连接卷，那么在active虚拟机宕机情况下，passive虚拟机可以直接变为active，由于两台虚拟机同时attach同一个cinder卷，因此不需要进行数据同步，这样对于集群应用有很大的便利性。  
如果没有mutil-attach功能，那么用户就需要克隆原始卷并将克隆卷attach到第二个实例。这样做的缺点是操作复杂，而且对原始卷的更改都不会显示在克隆卷中。  
### 问题描述Cinder-client目前在detach（）方法中只要求传入单个volume参数，这样从client端限制了主机可attach卷的数量，在外部的UI和内部组件的API上，也并没有支持mutil-attach。  Nova组件在代码层面上限制主机attach的数量，在attach动作发生时，nova会去检查改卷的状态，如果卷已经attach主机，那么就不能再attach其他主机。代码在nova/volume/cinder.py: check_attach()。  
### RO/RW情况分类Multi-Attach RO：  能够以正常方式连接卷（读写访问权限），然后以只读模式将该卷附加到另一个主机/服务器。问题在于，要求使用者（即Nova / KVM）知道如何在附加卷上设置只读模式和强制执行只读模式。  
Multi-Attach RW：  能够像通常那样附加卷（读写访问权限），然后将该卷附加到另一个主机/服务器。 在这种情况下，多个attachment之间没有区别。它们都被视为独立项目，并且都是读写卷。 
   目前，所有后端卷都将以读写（RW）模式attach，包括boot from volume模式。用户可以使用两个新的Cinder策略打开或关闭该功能：  volume:multiattach  volume:multiattach_bootable_volume  
###实现方式OpenStack’s multiattach Feature允许一个volume经iscsi/FC被attached到多个VM  
1.要启用将卷附加到多个主机或实例的功能，需要一个新的volume_attachment表来跟踪cinder卷的每个attachment。 将从volume表（attached_host，instance_uuid，mountpoint，attach_time，attach_mode）迁移现有列到新的表。 volume_attachment表将具有一个id（称为attachment_id），用于跟踪每个卷的各个attachment。 调用cinder API可返回的现有卷具有的attachment列表。 此attachment列表包含每个attachment的attachment_id。  
2.attachment_id是nova在发生detach动作时API传递给cinder，因此cinder知道要detach哪个attachment。 cinder API将支持默认attachment_id为None，如果只有一个attachment，它将尝试分离。  
3.执行分离（detach）动作时，如果cinder没有获得attachment_id，并且该卷只有1个attachment，那么分离该attachment的动作成功。 如果该卷有多个attachment，则cinder将返回Nova一个错误消息，指出该卷有多个attachment并需要attachment_id。  
4.卷表将被更新为包含一个名为'multiattach'的新列，该列是一个布尔标志，用于指示cinder该卷可以/不能连接多于一次。cinder API和cinderclient将支持在卷附加时设置该布尔标志。  
### Nova组件  
在Queens版本前，Nova尚未准备将单个Cinder卷附加到多个VM实例，即使卷本身允许该操作。默认情况下，libvirt假定所有磁盘都由一个guest虚拟机专用。如果要在实例之间共享磁盘，则需要在为该磁盘配置guest虚拟机XML时，通过设置磁盘的“可共享”标志来告诉libvirt。这样虚拟机管理程序不会尝试对磁盘进行独占锁定，并禁用所有I / O缓存，并且任何SELinux标签都允许所有域使用。  
如果hypervisor层本身不支持mutilattach，Nova应该拒绝attach请求，但是使用当前的API是不可能做到对hypervisor支持性的检查。但是这可以通过用户配置策略规则来解决。 例如，运行的云平台不支持multiattach，假设hypervisor层是vmware，那么用户可以配置策略来禁用cinder端的multiattach卷.  
如果是混合云，用户在尝试将卷附加到多个实例时，如果该compute节点的virt驱动程序不支持multiattach的计算机上的实例，那么attach请求将失败，并且nova-compute调用attachment_delete将删除通过nova-api创建的attachment。  
如果nova-api可以检查后端存储是否具备了mutilattach功能，那么用户可以通过API直接调用获取，但是nova还没有该API，所以只能通过手动配置cinder和尝试attach这两种方式。  

![2](/images/cinder_mutilattach/2.png)
###配置命令  
为了能够将卷附加到多个实例，需要在配置文件中将'multiattach'标志设置为'True'。   $ cinder type-create multiattach  $ cinder type-key multiattach set multiattach="<is> True"  要创建卷，需要使用之前创建的卷类型，如下所示：  $ cinder create <volume_size> --name <volume_name> --volume-type <volume_type_uuid>  详细配置可参考文档[3]  ###注意点  
1.在Queens版本中，mutil-attach实现了三种Driver：LVM，NetApp / SolidFire和Oracle ZFSSA。 可以查看驱动程序支持列表[4]，了解其他驱动程序何时添加支持的更新。另外支持mutilattach的卷不支持加密。  2.Multi-Attach 功能在 cinder microversion >= 3.50 版本可用，查看 stable/queens 的cinder版本。  ###性能/安全性影响1.可能的性能损失是在卷读取数据库对volume_attachment表的额外提取。 这是一个简单的外键表连接，并且是索引的。  2.在libvirt驱动程序中，磁盘被赋予一个共享的SELinux标签，因此磁盘不再具有sVirt SELinux隔离。  ###Horizon的更改Horizon组件为了支持mutil-attach所需的更改是：  1.当卷连接到多个主机时，改进Volumes表中Attached To列的格式。目前，该字段被设置为1，并且在连接多主机时不会缩放。  2.在卷创建对话框中添加一个新复选框，将卷标记为可共享多个实例。  3.修改“编辑卷attachment”对话框以允许将可共享卷附加到多个实例。  OpenStack基金会表示，新的SDS功能是云环境中最受欢迎的功能之一，该功能提供存储冗余。如果一个节点发生故障，另一个节点可接管并允许访问该卷。通常，有多台前端服务器连接到一台后端的高端服务器上，当一台前端服务器 down 掉，仍可以通过其他的前端服务器访问后端高端服务器。通过 Cinder的新功能，可以利用虚拟存储提供相同级别的高可用性，而不依赖昂贵的光纤通道存储阵列（共享存储）。  ###参考文献[1].https://specs.openstack.org/openstack/nova-specs/specs/queens/implemented/multi-attach-volume.html  [2],https://wiki.openstack.org/wiki/Cinder/blueprints/multi-attach-volume  [3],https://www.sunmite.com/openstack/cinder-multi-attch.html  [4],https://wiki.openstack.org/wiki/CinderSupportMatrix  
 