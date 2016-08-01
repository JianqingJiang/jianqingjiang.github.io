---
layout: post
title: OpenStack对接Ceph
description: "OpenStack对接Ceph"
tags: [OpenStack]
categories: [OpenStack]
---
###   环境：3台Controller的硬盘作为ceph存储，每台Controller上有3块硬盘，其余为Compute节点

## install clients

每个节点安装（要访问ceph集群的节点）

```
sudo yum install python-rbd 
sudo yum install ceph-common
```


## create pools

只需在一个ceph节点上操作即可

```
ceph osd pool create images 1024
ceph osd pool create vms 1024
ceph osd pool create volumes 1024
```

显示pool的状态

```
ceph osd lspools

ceph -w
3328 pgs: 3328 active+clean
```

## 创建用户

只需在一个ceph节点上操作即可


```
ceph auth get-or-create client.glance mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=images'
ceph auth get-or-create client.cinder mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=vms, allow rx pool=images'
nova使用cinder用户，就不单独创建了
```

## 拷贝ceph-ring

只需在一个ceph节点上操作即可

```
ceph auth get-or-create client.glance > /etc/ceph/ceph.client.glance.keyring
ceph auth get-or-create client.cinder > /etc/ceph/ceph.client.cinder.keyring
```


使用scp拷贝到其他节点

```
[root@controller1 ceph]# ls
ceph.client.admin.keyring   ceph.conf  tmpOZNQbH
ceph.client.cinder.keyring  rbdmap     tmpPbf5oV
ceph.client.glance.keyring  tmpJSCMCV  tmpUKxBQB
```


## 权限

更改文件的权限(所有客户端节点均执行)


```
sudo chown glance:glance /etc/ceph/ceph.client.glance.keyring
sudo chown cinder:cinder /etc/ceph/ceph.client.cinder.keyring
```


更改libvirt权限（只需在nova-compute节点上操作即可）

```
uuidgen
生成uuid

cat > secret.xml <<EOF
<secret ephemeral='no' private='no'>
  <uuid>457eb676-33da-42ec-9a8c-9293d545c337</uuid>
  <usage type='ceph'>
    <name>client.cinder secret</name>
  </usage>
</secret>
EOF
```

拷贝到所有compute节点


```
sudo virsh secret-define --file secret.xml
生成key
ceph auth get-key client.cinder > ./client.cinder.key
sudo virsh secret-set-value --secret adc522c4-237c-4cb8-8e39-682adcf4a830 --base64 $(cat ./client.cinder.key)
```

最后所有compute节点的client.cinder.key和secret.xml都是一样的  

记下之前生成的uuid 

```
adc522c4-237c-4cb8-8e39-682adcf4a830
```



##  配置Glance


在所有的controller节点上做如下更改

```
etc/glance/glance-api.conf


[DEFAULT]
...
default_store = rbd
...
[glance_store]
stores = rbd
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_chunk_size = 8
```


## 验证Glance 
在所有的controller节点上做如下更改



重启服务

```
openstack-service restart glance
```

查看状态

```
[root@controller1 ~]# openstack-service status glance
MainPID=23178 Id=openstack-glance-api.service ActiveState=active
MainPID=23155 Id=openstack-glance-registry.service ActiveState=active
```
```
[root@controller1 ~]# netstat -plunt | grep 9292
tcp        0      0 192.168.53.58:9292      0.0.0.0:*               LISTEN      23178/python2
tcp        0      0 192.168.53.23:9292      0.0.0.0:*               LISTEN      11435/haproxy
```

上传一个镜像  

```
[root@controller1 ~]# rbd ls images
ac5c334f-fbc2-4c56-bf48-47912693b692
```

这样就说明成功了


## 配置 Cinder


```
/etc/cinder/cinder.conf

[DEFAULT]
...
enabled_backends = ceph
...
[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1
glance_api_version = 2
rbd_user = cinder
rbd_secret_uuid = adc522c4-237c-4cb8-8e39-682adcf4a830

注意：ceph段在文本中要重新创建

```

##  验证Cinder

```
openstack-service restart cinder
创建一个云硬盘
[root@controller1 ~]# rbd ls volumes
volume-463c3495-1747-480f-974f-51ac6e1c5612
```

这样就成功了  



##  配置Nova

```
/etc/nova/nova.conf

[libvirt]
images_type = rbd
images_rbd_pool = vms
images_rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = cinder
rbd_secret_uuid = 457eb676-33da-42ec-9a8c-9293d545c337
disk_cachemodes="network=writeback"

inject_password = false
inject_key = false
inject_partition = -2
live_migration_flag="VIR_MIGRATE_UNDEFINE_SOURCE,VIR_MIGRATE_PEER2PEER,VIR_MIGRATE_LIVE,VIR_MIGRATE_PERSIST_DEST,VIR_MIGRATE_TUNNELLED"
```

## 配置Neutron网络

```
[root@compute1 ~]# ovs-vsctl add-br br-data
[root@compute1 ~]# ovs-vsctl add-port br-data enp7s0f0
[root@compute1 ~]# egrep -v "^$|^#" /etc/neutron/plugins/ml2/openvswitch_agent.ini
[ovs]
integration_bridge = br-int
bridge_mappings =default:br-data
enable_tunneling=False
[agent]
polling_interval = 2
l2_population = False
arp_responder = False
prevent_arp_spoofing = True
enable_distributed_routing = False
extensions =
drop_flows_on_start=False
[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```

磁盘总量如下，使用了3T*9=27T(ceph3备份)  

![1](/images/openstack_ceph/1.png) 

