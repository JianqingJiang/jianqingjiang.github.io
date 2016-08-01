---
layout: post
title: OpenStack增加Compute节点（并对接Ceph存储）
description: "OpenStack增加Compute节点（并对接Ceph存储）"
tags: [OpenStack]
categories: [OpenStack]
---
##  Add a compute node

### 安装包
```
yum install openstack-nova-compute
yum install openstack-neutron
yum install openstack-ceilometer-compute
yum install openstack-neutron-openvswitch
```

### 拷贝其他Compute节点的配置信息

```
scp /etc/nova/nova.conf 192.168.53.53:/etc/nova/nova.conf
scp /etc/neutron 192.168.53.53:/etc/neutron
scp -r /etc/ceilometer/ 192.168.53.53:/etc/ceilometer/
注意更改权限
```

###   启动服务


```
service openvswitch start
systemctl enable openvswitch
service libvirtd start
systemctl enable libvirtd
service neutron-openvswitch-agent start
systemctl enable neutron-openvswitch-agent
service openstack-ceilometer-compute start
systemctl enable openstack-ceilometer-compute
```

###   配置网桥

```
ovs-vsctl add-br br-data
ovs-vsctl add-port br-data enp7s0f0
```

### 对接Ceph

```
yum install ceph-common
把controller3上的根目录下的secret.xml拷贝到compute2上
sudo virsh secret-define --file secret.xml

生成key
ceph auth get-key client.cinder > ./client.cinder.key
sudo virsh secret-set-value --secret adc522c4-237c-4cb8-8e39-682adcf4a830 --base64 $(cat ./client.cinder.key)

重启服务
service openstack-nova-compute restart
systemctl enable openstack-nova-compute
```
![1](/images/openstack_add_compute/1.png) 

