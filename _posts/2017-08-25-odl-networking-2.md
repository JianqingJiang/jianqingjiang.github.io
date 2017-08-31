---
layout: post
title: ODL-Networking之深入理解（一）
description: "ODL-Networking之深入理解（一）"
tags: [OpenStack]
categories: [OpenStack]
---


###   opendaylight_v1与v2区别
在Neutron服务节点更改配置```openstack@network:~$ sudo vi /etc/neutron/plugins/ml2/ml2_conf.ini[ml2]type_drivers = flat,vxlantenant_network_types = vxlanmechanism_drivers = opendaylight [ml2_odl]password = adminusername = adminurl = http://192.168.51.110:8080/controller/nb/v2/neutron```安装odl-networking还是比较推荐使用源码安装，这样的话比较好控制。在使用  
python setup.py install  
的时候可能出现以下问题  
<pre><code>  
[root@controller neutron-lbaas-dashboard(keystone_admin)]# python setup.py installTraceback (most recent call last):  File "setup.py", line 29, in <module>    pbr=True)  File "/usr/lib64/python2.7/distutils/core.py", line 112, in setup    _setup_distribution = dist = klass(attrs)  File "/usr/lib/python2.7/site-packages/setuptools/dist.py", line 265, in __init__    self.fetch_build_eggs(attrs.pop('setup_requires'))  File "/usr/lib/python2.7/site-packages/setuptools/dist.py", line 289, in fetch_build_eggs    parse_requirements(requires), installer=self.fetch_build_egg  File "/usr/lib/python2.7/site-packages/pkg_resources.py", line 630, in resolve    raise VersionConflict(dist,req) # XXX put more info herepkg_resources.VersionConflict: (pbr 1.10.0 (/usr/lib/python2.7/site-packages), Requirement.parse('pbr>=2.0.0'))
</code></pre>
这种情况的话只需要把git branch调整为当前的openstack环境的版本就可以了  
###    配置步骤  
odl-networking安装完成之后需要同步数据库，在数据库中增加journal和maintenance数据，否则会报错  <pre><code>  
[root@localhost ~(keystone_admin)]# sudo su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutronINFO  [alembic.runtime.migration] Context impl MySQLImpl.INFO  [alembic.runtime.migration] Will assume non-transactional DDL.  Running upgrade for neutron ...INFO  [alembic.runtime.migration] Context impl MySQLImpl.INFO  [alembic.runtime.migration] Will assume non-transactional DDL.  OKINFO  [alembic.runtime.migration] Context impl MySQLImpl.INFO  [alembic.runtime.migration] Will assume non-transactional DDL.  Running upgrade for networking-odl ...INFO  [alembic.runtime.migration] Context impl MySQLImpl.INFO  [alembic.runtime.migration] Will assume non-transactional DDL.INFO  [alembic.runtime.migration] Running upgrade  -> b89a299e19f9, Initial odl db, branchpointINFO  [alembic.runtime.migration] Running upgrade b89a299e19f9 -> 247501328046, Start of odl expand branchINFO  [alembic.runtime.migration] Running upgrade 247501328046 -> 37e242787ae5, Opendaylight Neutron mechanism driver refactorINFO  [alembic.runtime.migration] Running upgrade 37e242787ae5 -> 703dbf02afde, Add journal maintenance tableINFO  [alembic.runtime.migration] Running upgrade 703dbf02afde -> 3d560427d776, add sequence number to journalINFO  [alembic.runtime.migration] Running upgrade b89a299e19f9 -> 383acb0d38a0, Start of odl contract branchINFO  [alembic.runtime.migration] Running upgrade 383acb0d38a0 -> fa0c536252a5, update opendayligut journal  OKINFO  [alembic.runtime.migration] Context impl MySQLImpl.INFO  [alembic.runtime.migration] Will assume non-transactional DDL.  Running upgrade for neutron-lbaas ...INFO  [alembic.runtime.migration] Context impl MySQLImpl.INFO  [alembic.runtime.migration] Will assume non-transactional DDL.  OK
</code></pre>    
这步骤操作完成之后需要重启neutron-server  ###  odl-networking数据库
数据库中含有journal和maintenance两张表  
![image](/images/odl-networking-2/1.png)  

journal表结构  
![image](/images/odl-networking-2/2.png)  

maintenance表结构   

![image](/images/odl-networking-2/3.png)  ###  复用代码
如果想使用这个脚本为自己的sdn控制器服务，而不是opendaylight的话，可以删除一些代码就可以使用，如果你的控制器并没有自己的数据库的话，那么就不需要这个odl-networking程序去进行数据库同步，可以注释掉一些代码（基于newton branch）
![image](/images/odl-networking-2/4.png)  
![image](/images/odl-networking-2/5.png)  