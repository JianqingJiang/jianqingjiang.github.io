---
layout: post
title: OpenStack中虚拟机一直在同一台物理机中孵化
description: "OpenStack中虚拟机一直在同一台物理机中孵化"
tags: [OpenStack]
categories: [OpenStack]
---

## 背景

OpenStack中NOVA服务在comupute2和compute3上面都是UP的，但是虚拟机起的都在compute2上面
我把compute3中的服务都卸载了，检查了nova.conf和neutron文件夹中的配置文件发现都没问题，我把libvirt文件也删除了，重新安装了KVM

![image](/images/openstack_rebalance/1.jpg)

![image](/images/openstack_rebalance/2.jpg)

发现问题仍然存在

## 调试


在nova的log中发现了如下log  
vi /var/log/nova/nova-conductor.log

```
2017-03-16 06:11:55.355 1511 ERROR nova.scheduler.utils [req-9779d696-34bd-442a-9b71-5e3a0ec8893f 5be92d6fb50e4c188f4991b1e7988c1b 1cac0a39dbb4473e85c836595a0e7673 - - -] [instance: 8c818ee0-36d8-450b-a61f-3744305d4244] Error from last host: compute3 (node compute3): [u'Traceback (most recent call last):\n', u'  File "/usr/lib/python2.7/site-packages/nova/compute/manager.py", line 1905, in _do_build_and_run_instance\n    filter_properties)\n', u'  File "/usr/lib/python2.7/site-packages/nova/compute/manager.py", line 2058, in _build_and_run_instance\n    instance_uuid=instance.uuid, reason=six.text_type(e))\n', u"RescheduledException: Build of instance 8c818ee0-36d8-450b-a61f-3744305d4244 was re-scheduled: Secret not found: no secret with matching uuid '5b059071-1aff-4b72-bc5f-0122a7d6c1df'\n"]

```


## 原因

我把libvirt文件删除了之后，里面有秘钥文件，也被删除了，我从compute2中把文件拷贝到compute3的libvirt对应目录中，重启服务就OK了


![image](/images/openstack_rebalance/3.jpg)

![image](/images/openstack_rebalance/4.jpg)

