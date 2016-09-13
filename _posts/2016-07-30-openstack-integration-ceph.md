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


[DEFAULT]default_store = rbdshow_image_direct_url=Falsebind_host=controller1bind_port=9292workers=24backlog=4096image_cache_dir=/var/lib/glance/image-cacheregistry_host=controller1registry_port=9191registry_client_protocol=httpdebug=Falseverbose=Truelog_file=/var/log/glance/api.loglog_dir=/var/log/glanceuse_syslog=Falsesyslog_log_facility=LOG_USERuse_stderr=True[database]connection=mysql+pymysql://glance:a9717e6e21384389@controller:3305/glanceidle_timeout=3600[glance_store]os_region_name=RegionOnestores = rbddefault_store = rbdrbd_store_pool = imagesrbd_store_user = glancerbd_store_ceph_conf = /etc/ceph/ceph.confrbd_store_chunk_size = 8[image_format][keystone_authtoken]auth_uri=http://controller:5000/v2.0identity_uri=http://controller:35357admin_user=glanceadmin_password=79c14ca2ba51415badmin_tenant_name=services[matchmaker_redis][matchmaker_ring][oslo_concurrency][oslo_messaging_amqp][oslo_messaging_qpid][oslo_messaging_rabbit]rabbit_host=controller1,controller2,controller3rabbit_hosts=controller1:5672,controller2:5672,controller3:5672[oslo_policy][paste_deploy]flavor=keystone[store_type_location_strategy][task][taskflow_executor]
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
enabled_backends = ceph
glance_host = controller
enable_v1_api = True
enable_v2_api = True
host = controller1
storage_availability_zone = nova
default_availability_zone = nova
auth_strategy = keystone
#enabled_backends = lvm
osapi_volume_listen = controller1
osapi_volume_workers = 24
nova_catalog_info = compute:nova:publicURL
nova_catalog_admin_info = compute:nova:adminURL
debug = False
verbose = True
log_dir = /var/log/cinder
notification_driver =messagingv2
rpc_backend = rabbit
control_exchange = openstack
api_paste_config=/etc/cinder/api-paste.ini
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
volume_backend_name=ceph
[BRCD_FABRIC_EXAMPLE]
[CISCO_FABRIC_EXAMPLE]
[cors]
[cors.subdomain]
[database]
connection = mysql+pymysql://cinder:02d2f2f82467400d@controller:3305/cinder
[fc-zone-manager]
[keymgr]
[keystone_authtoken]
auth_uri = http://controller:5000
identity_uri = http://controller:35357
admin_user = cinder
admin_password = 7281b7bf47044f96
admin_tenant_name = services
memcache_servers = 127.0.0.1:11211  
token_cache_time = 3600
cache = true  
[matchmaker_redis]
[matchmaker_ring]
[oslo_concurrency]
[oslo_messaging_amqp]
[oslo_messaging_qpid]
[oslo_messaging_rabbit]
amqp_durable_queues = False
kombu_ssl_keyfile =
kombu_ssl_certfile =
kombu_ssl_ca_certs =
rabbit_host = controller1,controller2,controller3
rabbit_port = 5672
rabbit_hosts = controller1:5672,controller2:5672,controller3:5672
rabbit_use_ssl = False
rabbit_userid = guest
rabbit_password = guest
rabbit_virtual_host = /
rabbit_ha_queues = False
heartbeat_timeout_threshold = 0
heartbeat_rate = 2
[oslo_middleware]
[oslo_policy]
[oslo_reports]
[profiler]
[lvm]
iscsi_helper=lioadm
volume_group=cinder-volumes
iscsi_ip_address=192.168.56.251
volume_driver=cinder.volume.drivers.lvm.LVMVolumeDriver
volumes_dir=/var/lib/cinder/volumes
iscsi_protocol=iscsi
volume_backend_name=lvm

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


[DEFAULT]internal_service_availability_zone=internaldefault_availability_zone=novause_ipv6=Falsenotify_api_faults=Falsestate_path=/var/lib/novareport_interval=10compute_manager=nova.compute.manager.ComputeManagerservice_down_time=60rootwrap_config=/etc/nova/rootwrap.confvolume_api_class=nova.volume.cinder.APIauth_strategy=keystoneallow_resize_to_same_host=Falseheal_instance_info_cache_interval=60reserved_host_memory_mb=512network_api_class=nova.network.neutronv2.api.APIforce_snat_range =0.0.0.0/0metadata_host=192.168.56.200dhcp_domain=novalocalsecurity_group_api=neutroncompute_driver=libvirt.LibvirtDrivervif_plugging_is_fatal=Truevif_plugging_timeout=300firewall_driver=nova.virt.firewall.NoopFirewallDriverforce_raw_images=Truedebug=Falseverbose=Truelog_dir=/var/log/novause_syslog=Falsesyslog_log_facility=LOG_USERuse_stderr=Truenotification_topics=notificationsrpc_backend=rabbitvncserver_proxyclient_address=compute1vnc_keymap=en-ussql_connection=mysql+pymysql://nova:8a5033019dcf46c7@192.168.56.200/novavnc_enabled=Trueimage_service=nova.image.glance.GlanceImageServicelock_path=/var/lib/nova/tmpvncserver_listen=0.0.0.0novncproxy_base_url=http://192.168.56.200:6080/vnc_auto.html[api_database][barbican][cells][cinder]catalog_info=volumev2:cinderv2:publicURL[conductor][cors][cors.subdomain][database][ephemeral_storage_encryption][glance]api_servers=192.168.56.200:9292[guestfs][hyperv][image_file_url][ironic][keymgr][keystone_authtoken][libvirt]images_type = rbdimages_rbd_pool = vmsimages_rbd_ceph_conf = /etc/ceph/ceph.confrbd_user = cinderrbd_secret_uuid = 5b059071-1aff-4b72-bc5f-0122a7d6c1dfdisk_cachemodes="network=writeback"inject_password = falseinject_key = falseinject_partition = -2live_migration_flag="VIR_MIGRATE_UNDEFINE_SOURCE,VIR_MIGRATE_PEER2PEER,VIR_MIGRATE_LIVE,VIR_MIGRATE_PERSIST_DEST,VIR_MIGRATE_TUNNELLED"virt_type=kvminject_password=Falseinject_key=Falseinject_partition=-1live_migration_uri=qemu+tcp://nova@%s/systemcpu_mode=host-modelvif_driver=nova.virt.libvirt.vif.LibvirtGenericVIFDriver[matchmaker_redis][matchmaker_ring][metrics][neutron]url=http://192.168.56.200:9696admin_username=neutronadmin_password=0cccbadebaa14569admin_tenant_name=servicesregion_name=RegionOneadmin_auth_url=http://192.168.56.200:5000/v2.0auth_strategy=keystoneovs_bridge=br-intextension_sync_interval=600timeout=30default_tenant_id=default[osapi_v21][oslo_concurrency][oslo_messaging_amqp][oslo_messaging_qpid][oslo_messaging_rabbit]amqp_durable_queues=Falsekombu_reconnect_delay=1.0rabbit_host=controller1,controller2,controller3rabbit_port=5672rabbit_hosts=controller1:5672,controller2:5672,controller3:5672rabbit_use_ssl=Falserabbit_userid=guestrabbit_password=guestrabbit_virtual_host=/rabbit_ha_queues=Falseheartbeat_timeout_threshold=0heartbeat_rate=2[oslo_middleware][rdp][serial_console][spice][ssl][trusted_computing][upgrade_levels][vmware][vnc][workarounds][xenserver][zookeeper]
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

