---
layout: post
title: OpenStack High Availability
description: "OpenStack High Availability"
tags: [OpenStack]
categories: [OpenStack]
---
# Controller
### 基础配置  
```
# 主机名ip映射关系
[root@controller1 rabbitmq(keystone_admin)]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.53.58 controller1
192.168.53.67 controller2
192.168.53.68 controller3
192.168.53.92 compute1```


```
yum update  
yum install -y net-tools  
yum install -y centos-release-openstack-liberty  
yum install openstack-packstack  
```

```
vi /etc/profile
export LC_CTYPE=en_US.UTF-8
```


使用redhat的快速安装工具，CONFIG_COMPUTE_HOSTS只需要在一个controller上填写即可，不需要所有controller上都写上

```
packstack --gen-answer-file a.txt
```

```
vim a.txt
CONFIG_DEFAULT_PASSWORD=bnc
CONFIG_MANILA_INSTALL=n
CONFIG_CEILOMETER_INSTALL=n
CONFIG_NAGIOS_INSTALL=n
CONFIG_SWIFT_INSTALL=n
CONFIG_PROVISION_DEMO=n

CONFIG_NEUTRON_ML2_TYPE_DRIVERS=vlan
CONFIG_NEUTRON_ML2_TENANT_NETWORK_TYPES=vlan

CONFIG_NEUTRON_ML2_VLAN_RANGES=default:13:2000       # 配置vlan 范围
CONFIG_NEUTRON_OVS_BRIDGE_MAPPINGS=default:br-data
CONFIG_NEUTRON_OVS_BRIDGE_IFACES=br-data:enp7s0f0

CONFIG_COMPUTE_HOSTS=192.168.53.92                 # 计算节点都写在这里
```

```
packstack --answer-file a.txt    # 开始安装
```
安装完成  

![install](/images/openstack_ha/1.png)


###  删除一个服务


```
yum list | grep -i swift
openstack-service stop swift
openstack-service status swift
yum remove openstack-swift
rm -rf /etc/swift/
openstack service list
openstack service delete f18a5683261e496584a9d64d4d8f8ec1  #id号

```

###  修改


```
vi /etc/cinder/cinder.conf
auth_uri = http://<你的ip>:5000     # 去掉/v2.0

openstack-service restart cinder
```


##  High Availability


* ###   RabbitMQ HA(不特别标注的话，每个节点都需要)


```
[root@controller1 ~(keystone_admin)]# ll /var/lib/rabbitmq/ -al   # 查找.erlang.cookie
total 12
drwxr-x---.  3 rabbitmq rabbitmq   40 Jul  2 02:54 .
drwxr-xr-x. 46 root     root     4096 Jul  2 04:45 ..
-r--------.  1 rabbitmq rabbitmq   20 Jul  2 00:00 .erlang.cookie
drwxr-xr-x.  4 rabbitmq rabbitmq 4096 Jul  2 02:54 mnesia


scp -rp /var/lib/rabbitmq/.erlang.cookie  controller2:/var/lib/rabbitmq/
scp -rp /var/lib/rabbitmq/.erlang.cookie  controller3:/var/lib/rabbitmq/

# 校验所有节点的 文件内容是否一致
md5sum /var/lib/rabbitmq/.erlang.cookie

#查看RabbitMQ状态
service rabbitmq-server status

#重启RabbitMQ
service rabbitmq-server restart
```

```
#查看rabbitmq状态
rabbitmqctl cluster_statusCluster status of node rabbit@rabbit1 ...[{nodes,[{disc,[rabbit@rabbit1]}]},{running_nodes,[rabbit@rabbit1]}]...done.
``````rabbitmqctl stop_appStopping node rabbit@rabbit2 ...done.
rabbitmqctl join-cluster rabbit@controller1Clustering node rabbit@rabbit2 with [rabbit@controller] ...done.
rabbitmqctl start_appStarting node rabbit@rabbit2 ...done.


[root@controller2 ~(keystone_admin)]# rabbitmqctl cluster_status
Cluster status of node rabbit@controller2 ...
[{nodes,[{disc,[rabbit@controller1,rabbit@controller2,rabbit@controller3]}]},
 {running_nodes,[rabbit@controller1,rabbit@controller3,rabbit@controller2]},
 {cluster_name,<<"rabbit@controller1">>},
 {partitions,[]},
 {alarms,[{rabbit@controller1,[]},
          {rabbit@controller3,[]},
          {rabbit@controller2,[]}]}]

```

```
#这条命令在任意节点上运行一次就可以
rabbitmqctl set_policy ha-all "^." '{"ha-mode":"all"}'


#这条命令在任意节点可以查看上一步的配置
[root@controller2 ~(keystone_admin)]# rabbitmqctl list_policies
Listing policies ...
/	ha-all	all	^.	{"ha-mode":"all"}	0
```


```
启动管理插件[root@controller001 ~(keystone_admin)]# find / -name rabbitmq-plugins# 启用插件
[root@controller2 ~(keystone_admin)]# /usr/sbin/rabbitmq-plugins enable rabbitmq_management
The following plugins have been enabled:
  mochiweb
  webmachine
  rabbitmq_web_dispatch
  amqp_client
  rabbitmq_management_agent
  rabbitmq_management

Applying plugin configuration to rabbit@controller2... started 6 plugins.
```

```默认web ui url：http://server-name:15672默认user/pass: guest/guest
```
rabbitmq界面  
![界面](/images/openstack_ha/2.png)###   注意事项  * 为了防止数据丢失的发生，在任何情况下都应该保证至少有一个 node 是采用磁盘node 方式。RabbitMQ 在很多情况下会阻止创建仅有内存 node 的 cluster ，但是如果你通过手动将 cluster 中的全部磁盘 node 都停止掉或者强制 reset 所有的磁盘node 的方式间接导致生成了仅有内存 node 的 cluster ，RabbitMQ 无法阻止你。你这么做本身是很不明智的，因为会导致你的数据非常容易丢失。  
* 当整个 cluster 不能工作了，最后一个失效的 node 必须是第一个重新开始工作的那一个。如果这种情况得不到满足，所有 node 将会为最后一个磁盘 node 的恢复等待 30秒。如果最后一个离线的 node 无法重新上线，我们可以通过命令 forget_cluster_node将其从 cluster 中移除 - 具体参考 rabbitmqctl 的使用手册。  




