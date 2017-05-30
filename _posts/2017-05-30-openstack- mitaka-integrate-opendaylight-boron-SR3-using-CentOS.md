---
layout: post
title: OpenStack Mitaka版本对接Opendaylight Boron（硼）-SR3（using CentOS）
description: "OpenStack Mitaka版本对接Opendaylight Boron（硼）-SR3（using CentOS）"
tags: [SDN]
categories: [SDN]
---


由于ODL社区的文档更新实在是很慢，而且文档很多都是用DevStack或者是Ubuntu的环境，导致稍微版本更新一下，或者底层的OS换一下，参考的文档便不再适用了，由此还需要中文文档可以及时更新。

###  实验环境：
系统：CentOS7
OpenStack：Mitaka Stable版本，一台物理机起controller+network+compute ，一台物理机起compute  
ODL:Opendaylight Boron（硼）-SR3版本,部署在单独的物理机  


### 安装OpenStack:
如何安装OpenStack不在本文档的讨论范畴之内，可参考该官方文档，网络类型选择self-service就可以了
https://docs.openstack.org/mitaka/install-guide-rdo/

### 如何安装ODL：
由于ODL的安装可以下载编译好的代码，对新手已经有很大的帮助了，但是由于ODL的feature再不断的更新，使用氦版本的文档还是有问题
http://www.sdnlab.com/1931.html
上面的链接文档还是可以借鉴，需要先安装JDK版本，我安装了1.8版本，设置完环境变量之后就可以使用ODL了

<pre><code>
# unzip distribution-karaf-0.5.3-Boron-SR3.zip
# cd distribution-karaf-0.5.3-Boron-SR3/
# cd bin
# ./karaf
</code></pre>

硼版本已经不会有"线程异常且No route to host错误"这个问题了
在ODL shell中, 需要按照顺序安装下列的features:

```
opendaylight-user@root>feature:install odl-ovsdb-openstack
opendaylight-user@root>feature:install odl-dlux-core
opendaylight-user@root>feature:install odl-dlux-all
```
使用下列命令检查上述的features有没有正确的安装
```
opendaylight-user@root>feature:list | grep dlux
  < showing the list of features installed … >
opendaylight-user@root>feature:list | grep openstack
  < showing the list of features installed … >
opendaylight-user@root>
```

临时关闭防火墙和selinux

```
setenforce 0
systemctl stop firewalld
```

安装完成之后可以使用下列命令在任意节点获取到ODL上面的对接OpenStack API的信息：

<pre><code>
openstack@controller:~$ curl -u admin:admin http://192.168.51.110:8080/controller/nb/v2/neutron/networks
{
   "networks" : [ ]
}openstack@controller:~$
</code></pre>

到这里ODL上设置就基本完成了  
注意：请按照一定的顺序安装，安装顺序不合理的话，会导致后面Web界面无法访问！且记录遇到的一个问题：在没有按照顺序安装组件的情况下，无法登录进入ODL主界面。解决方法是通过logout退出karaf平台，进入bin目录：cd bin，执行./karaf clean，再次重复上面的安装组件操作。  


###OpenStack对接ODL  

### 清理OpenStack环境：
使用下列命令关闭neutron-server
```
openstack@controller:~$ sudo service neutron-server stop
```
在OpenStack上所有节点移除openvswitch-agent
```
openstack@network:~$ systemctl stop neutron-openvswitch-agent
openstack@network:~$ systemctl disable neutron-openvswitch-agent
openstack@network:~$ sudo rm -rf /var/log/openvswitch/
openstack@network:~$ sudo rm -rf /etc/openvswitch/conf.db
openstack@network:~$ sudo mkdir /var/log/openvswitch/
openstack@network:~$ sudo service openvswitch-switch start
openvswitch-switch start/running
openstack@network:~$ sudo ovs-vsctl show
2c63096f-74f3-46eb-9904-00305ef84106
    ovs_version: "2.5.0"
openstack@network:~$
```

让每个节点上的ovs连接ODL,没有br-int的话重新新建一个
```
sudo ovs-vsctl set-manager tcp:192.168.51.110:6640
ovs-vsctl set-controller br-int "tcp:192.168.51.110:6633"
```
然后在ODL的界面上就可以显示2个OVS已经连上来了
ODL的网页URL跟氦版本也有所不同
http://$your_odl_ip:8181/index.html
登录名/密码都是admin

![image](/images/openstack_integrate_odl/1.png)

如果需要上外网的话，需要增加br-ex
```
openstack@network:~$ sudo ovs-vsctl add-br br-ex
openstack@network:~$ sudo ovs-vsctl add-port br-ex eth0
openstack@network:~$ sudo ovs-vsctl show
2c63096f-74f3-46eb-9904-00305ef84106
    Manager "tcp:192.168.51.110:6640"
        is_connected: true
    Bridge br-ex
        Port br-ex
            Interface br-ex
                type: internal
        Port "eth3"
            Interface "eth0"
    Bridge br-int
        Controller "tcp:192.168.51.110:6653"
            is_connected: true
        fail_mode: secure
        Port br-int
            Interface br-int
                type: internal
    ovs_version: "2.5.0"
openstack@network:~$
```

在Neutron服务节点更改配置

```
openstack@network:~$ sudo vi /etc/neutron/plugins/ml2/ml2_conf.ini
[ml2]
type_drivers = flat,vxlan
tenant_network_types = vxlan
mechanism_drivers = opendaylight 

[ml2_odl]
password = admin
username = admin
url = http://192.168.51.110:8080/controller/nb/v2/neutron
```

在OpenStack Controller节点重新配置数据库  

<pre><code>
openstack@controller:~$ source ./admin_openrc.sh 
openstack@controller:~$ mysql -u root -pmysqlpassword
MariaDB [(none)]> DROP DATABASE neutron;
Query OK, 157 rows affected (1.76 sec)

MariaDB [(none)]> CREATE DATABASE neutron;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
Query OK, 0 rows affected (0.05 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'NEUTRON_DBPASS';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> exit
Bye
openstack@controller:$ rm -rf /etc/neutron/plugin.ini
openstack@controller:$ ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
openstack@controller:$ su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
</code></pre>

### 安装networking-odl
ODL现在已经不用在ml2文件下放入一个python脚本了，目前搞了一个专门的项目叫netwoking-odl，安装一下然后改下配置就可以了。从最新的Neutron代码中，已经发现了诸如原来的opendaylight和其他一些sdn Plugin，已经开始从项目中移除，统一命名为诸如networking-xxxx之类的独立项目。
```
openstack@controller:$ yum install python-pip -y 
openstack@controller:$ sudo pip install networking-odl
```
另一种安装方式如下
```
openstack@controller:$ git clone https://github.com/openstack/networking-odl -b stable/mitaka
openstack@controller:$ cd networking-odl/
openstack@controller:$ ./setup.py
```
文档说明安装完成之后才可以启动neutron-server,否则会出现如下的报错，说明OpenStack找不到ODL插件
<pre><code>
2017-05-26 22:57:27.434 5518 INFO neutron.common.config [-] Logging enabled!
2017-05-26 22:57:27.435 5518 INFO neutron.common.config [-] /usr/bin/neutron-server version 8.3.0
2017-05-26 22:57:27.436 5518 INFO neutron.common.config [-] Logging enabled!
2017-05-26 22:57:27.436 5518 INFO neutron.common.config [-] /usr/bin/neutron-server version 8.3.0
2017-05-26 22:57:27.454 5518 INFO neutron.manager [-] Loading core plugin: neutron.plugins.ml2.plugin.Ml2Plugin
2017-05-26 22:57:27.756 5518 INFO neutron.plugins.ml2.managers [-] Configured type driver names: ['flat', 'vxlan']
2017-05-26 22:57:27.759 5518 INFO neutron.plugins.ml2.drivers.type_flat [-] Allowable flat physical_network names: ['provider']
2017-05-26 22:57:27.765 5518 INFO neutron.plugins.ml2.managers [-] Loaded type driver names: ['flat', 'vxlan']
2017-05-26 22:57:27.766 5518 INFO neutron.plugins.ml2.managers [-] Registered types: ['flat', 'vxlan']
2017-05-26 22:57:27.766 5518 INFO neutron.plugins.ml2.managers [-] Tenant network_types: ['vxlan']
2017-05-26 22:57:27.766 5518 INFO neutron.plugins.ml2.managers [-] Configured extension driver names: []
2017-05-26 22:57:27.767 5518 INFO neutron.plugins.ml2.managers [-] Loaded extension driver names: []
2017-05-26 22:57:27.767 5518 INFO neutron.plugins.ml2.managers [-] Registered extension drivers: []
2017-05-26 22:57:27.767 5518 INFO neutron.plugins.ml2.managers [-] Configured mechanism driver names: ['opendaylight']
2017-05-26 22:57:27.768 5518 ERROR stevedore.extension [-] Could not load 'opendaylight': No module named opendaylight.driver
2017-05-26 22:57:27.768 5518 ERROR stevedore.extension [-] No module named opendaylight.driver
2017-05-26 22:57:27.768 5518 ERROR stevedore.extension Traceback (most recent call last):
2017-05-26 22:57:27.768 5518 ERROR stevedore.extension   File "/usr/lib/python2.7/site-packages/stevedore/extension.py", line 163, in _load_plugins
2017-05-26 22:57:27.768 5518 ERROR stevedore.extension     verify_requirements,
2017-05-26 22:57:27.768 5518 ERROR stevedore.extension   File "/usr/lib/python2.7/site-packages/stevedore/named.py", line 123, in _load_one_plugin
2017-05-26 22:57:27.768 5518 ERROR stevedore.extension     verify_requirements,
2017-05-26 22:57:27.768 5518 ERROR stevedore.extension   File "/usr/lib/python2.7/site-packages/stevedore/extension.py", line 184, in _load_one_plugin
2017-05-26 22:57:27.768 5518 ERROR stevedore.extension     plugin = ep.resolve()
2017-05-26 22:57:27.768 5518 ERROR stevedore.extension   File "/usr/lib/python2.7/site-packages/pkg_resources/__init__.py", line 2235, in resolve
2017-05-26 22:57:27.768 5518 ERROR stevedore.extension     module = __import__(self.module_name, fromlist=['__name__'], level=0)
2017-05-26 22:57:27.768 5518 ERROR stevedore.extension ImportError: No module named opendaylight.driver
</code></pre>

但是我在CentOS7下安装完了之后还是有上述报错，neutron-server依然起不来  
我的默认的ml2 plugins像openvswitch是安装下如下路径下的/usr/lib/python2.7/site-packages/neutron/plugins/ml2/drivers/openvswitch/  
而且我的networking_odl安装包已经安装在如下路径下/usr/lib/python2.7/site-packages/networking_odl/ml2/  
谷歌了几天之后，得出以下解决方案  
我需要把/usr/lib/python2.7/site-packages/networking_odl-2.0.1.dev15-py2.7.egg-info/entry_points.txt  
里面的内容合并到/usr/lib/python2.7/site-packages/neutron-8.3.0-py2.7.egg-info/entry_points.txt中，终于neutron-server停止抱怨了  
合并完成的entry_points.txt如下  

<pre><code>
[neutron.ml2.mechanism_drivers]
opendaylight = networking_odl.ml2.mech_driver:OpenDaylightMechanismDriver
opendaylight_v2 = networking_odl.ml2.mech_driver_v2:OpenDaylightMechanismDriver
fake_agent = neutron.tests.unit.plugins.ml2.drivers.mech_fake_agent:FakeAgentMechanismDriver
l2population = neutron.plugins.ml2.drivers.l2pop.mech_driver:L2populationMechanismDriver
linuxbridge = neutron.plugins.ml2.drivers.linuxbridge.mech_driver.mech_linuxbridge:LinuxbridgeMechanismDriver
logger = neutron.tests.unit.plugins.ml2.drivers.mechanism_logger:LoggerMechanismDriver
macvtap = neutron.plugins.ml2.drivers.macvtap.mech_driver.mech_macvtap:MacvtapMechanismDriver
openvswitch = neutron.plugins.ml2.drivers.openvswitch.mech_driver.mech_openvswitch:OpenvswitchMechanismDriver
sriovnicswitch = neutron.plugins.ml2.drivers.mech_sriov.mech_driver.mech_driver:SriovNicSwitchMechanismDriver
test = neutron.tests.unit.plugins.ml2.drivers.mechanism_test:TestMechanismDriver
</code></pre>

### 验证

在OpenStack中创建网络后，执行下面命令
```
[root@controller ~]#  curl -u admin:admin http://192.168.51.110:8080/controller/nb/v2/neutron/networks
{
   "networks" : [ {
      "id" : "6a678525-2853-419a-8a4e-26a714644cc6",
      "admin_state_up" : true,
      "status" : "ACTIVE",
      "tenant_id" : "93545f587b074e50ad70571e2d9df7a6",
      "name" : "net1",
      "shared" : false,
      "router:external" : false,
      "provider:network_type" : "vxlan",
      "provider:segmentation_id" : "80",
      "segments" : [ ]
   } ]
}[root@controller ~]#
```
ODL已经可以获取到网络信息
创建2台虚机，可以成功获取到IP地址，并且可以相互访问
![image](/images/openstack_integrate_odl/2.png)

OpenStack与ODL安装至此就成功了。


参考文档
https://docs.openstack.org/developer/networking-odl/installation.html  
https://wiki.opendaylight.org/view/OpenStack_and_OpenDaylight  
http://docs.opendaylight.org/en/stable-boron/submodules/netvirt/docs/openstack-guide/openstack-with-netvirt.html#installing-opendaylight-on-an-existing-openstack