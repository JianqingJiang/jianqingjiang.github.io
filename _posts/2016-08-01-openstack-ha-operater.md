---
layout: post
title: OpenStack HA Operate
description: "OpenStack HA Operater"
tags: [OpenStack]
categories: [OpenStack]
---

###   首先看服务状态

```
[root@controller1 ~(keystone_admin)]# openstack-service status
MainPID=23517 Id=neutron-dhcp-agent.service ActiveState=active
MainPID=23510 Id=neutron-l3-agent.service ActiveState=active
MainPID=23522 Id=neutron-metadata-agent.service ActiveState=active
MainPID=23823 Id=neutron-openvswitch-agent.service ActiveState=active
MainPID=23544 Id=neutron-server.service ActiveState=active
MainPID=23599 Id=openstack-ceilometer-alarm-evaluator.service ActiveState=active
MainPID=23600 Id=openstack-ceilometer-alarm-notifier.service ActiveState=active
MainPID=23506 Id=openstack-ceilometer-api.service ActiveState=active
MainPID=23598 Id=openstack-ceilometer-central.service ActiveState=active
MainPID=23596 Id=openstack-ceilometer-collector.service ActiveState=active
MainPID=23597 Id=openstack-ceilometer-notification.service ActiveState=active
MainPID=23593 Id=openstack-cinder-api.service ActiveState=active
MainPID=23595 Id=openstack-cinder-scheduler.service ActiveState=active
MainPID=23695 Id=openstack-cinder-volume.service ActiveState=active
MainPID=23536 Id=openstack-glance-api.service ActiveState=active
MainPID=23531 Id=openstack-glance-registry.service ActiveState=active
MainPID=0 Id=openstack-losetup.service ActiveState=active
MainPID=24557 Id=openstack-nova-api.service ActiveState=active
MainPID=23826 Id=openstack-nova-cert.service ActiveState=active
MainPID=24160 Id=openstack-nova-conductor.service ActiveState=active
MainPID=23825 Id=openstack-nova-consoleauth.service ActiveState=active
MainPID=23658 Id=openstack-nova-novncproxy.service ActiveState=active
MainPID=23824 Id=openstack-nova-scheduler.service ActiveState=active
```

###   一般的问题重启相应服务

```
service keepalived restart
service httpd restart
openstack-service restart
```

###   检查服务状态

```
openstack-status
openstack-service status
rabbitmqctl cluster_status```


###    通过vip测试keystone数据库连接

```
mysql -ukeystone_admin -pf15c18d2db7a4804 -h controller -P 3305
key在下面配置文件中
/etc/keystone/keystone.conf
```

###    查看数据库集群情况

``` 
MariaDB [(none)]> show status like 'wsrep%';+------------------------------+--------------------------------------+| Variable_name                | Value                                |+------------------------------+--------------------------------------+| wsrep_local_state_uuid       | b5b8da17-40c4-11e6-a3f9-1fe593f04af7 || wsrep_protocol_version       | 5                                    || wsrep_last_committed         | 17757240                             || wsrep_replicated             | 7269820                              || wsrep_replicated_bytes       | 3773747388                           || wsrep_repl_keys              | 33972870                             || wsrep_repl_keys_bytes        | 438989071                            || wsrep_repl_data_bytes        | 2869489837                           || wsrep_repl_other_bytes       | 0                                    || wsrep_received               | 146596                               || wsrep_received_bytes         | 28385685                             || wsrep_local_commits          | 7269288                              || wsrep_local_cert_failures    | 517                                  || wsrep_local_replays          | 0                                    || wsrep_local_send_queue       | 0                                    || wsrep_local_send_queue_avg   | 0.001016                             || wsrep_local_recv_queue       | 0                                    || wsrep_local_recv_queue_avg   | 0.201609                             || wsrep_local_cached_downto    | 17535412                             || wsrep_flow_control_paused_ns | 14824230237                          || wsrep_flow_control_paused    | 0.000010                             || wsrep_flow_control_sent      | 9                                    || wsrep_flow_control_recv      | 66                                   || wsrep_cert_deps_distance     | 1.040951                             || wsrep_apply_oooe             | 0.248419                             || wsrep_apply_oool             | 0.000113                             || wsrep_apply_window           | 1.287288                             || wsrep_commit_oooe            | 0.000000                             || wsrep_commit_oool            | 0.000113                             || wsrep_commit_window          | 1.035603                             || wsrep_local_state            | 4                                    || wsrep_local_state_comment    | Synced                               || wsrep_cert_index_size        | 67                                   || wsrep_causal_reads           | 0                                    || wsrep_cert_interval          | 0.294778                             || wsrep_incoming_addresses     | controller1:3306,controller2:3306,   || wsrep_cluster_conf_id        | 216                                  || wsrep_cluster_size           | 3                                    || wsrep_cluster_state_uuid     | b5b8da17-40c4-11e6-a3f9-1fe593f04af7 || wsrep_cluster_status         | Primary                              || wsrep_connected              | ON                                   || wsrep_local_bf_aborts        | 34                                   || wsrep_local_index            | 0                                    || wsrep_provider_name          | Galera                               || wsrep_provider_vendor        | Codership Oy <info@codership.com>    || wsrep_provider_version       | 3.5(rXXXX)                           || wsrep_ready                  | ON                                   || wsrep_thread_count           | 9                                    |+------------------------------+--------------------------------------+48 rows in set (0.16 sec)
```
 
 
###   检查mongodb状态（ceilometer服务）


```
[root@controller1 ~(keystone_admin)]# mongo -host controller1
MongoDB shell version: 2.6.11
connecting to: controller1:27017/test
s1:PRIMARY> rs.status()
{
	"set" : "s1",
	"date" : ISODate("2016-07-31T01:59:46Z"),
	"myState" : 1,
	"members" : [
		{
			"_id" : 0,
			"name" : "controller1:27017",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 2421182,
			"optime" : Timestamp(1469930372, 2),
			"optimeDate" : ISODate("2016-07-31T01:59:32Z"),
			"electionTime" : Timestamp(1469880927, 1),
			"electionDate" : ISODate("2016-07-30T12:15:27Z"),
			"self" : true
		},
		{
			"_id" : 1,
			"name" : "controller2:27017",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 49387,
			"optime" : Timestamp(1469930372, 2),
			"optimeDate" : ISODate("2016-07-31T01:59:32Z"),
			"lastHeartbeat" : ISODate("2016-07-31T01:59:44Z"),
			"lastHeartbeatRecv" : ISODate("2016-07-31T01:59:44Z"),
			"pingMs" : 0,
			"syncingTo" : "controller1:27017"
		},
		{
			"_id" : 2,
			"name" : "controller3:27017",
			"health" : 1,
			"state" : 7,
			"stateStr" : "ARBITER",
			"uptime" : 49464,
			"lastHeartbeat" : ISODate("2016-07-31T01:59:44Z"),
			"lastHeartbeatRecv" : ISODate("2016-07-31T01:59:46Z"),
			"pingMs" : 0
		}
	],
	"ok" : 1
}
s1:PRIMARY>
```

vnc界面调试优化

```
vim /etc/nova/nova.conf
memcached_servers=controller1:11211,controller2:11211,controller3:11211
service openstack-nova-consoleauth restart
```


