---
layout: post
title: OpenStack对接ODL控制器之ODL-Networking模块详解
description: "OpenStack对接ODL控制器之ODL-Networking模块详解"
tags: [OpenStack]
categories: [OpenStack]
---
OpenStack支持多种SDN控制器的对接，之前的对接方案是将一个mechanism driver的一个python文件放在neutron ml2的目录下面，使得OpenStack对neutron的操作可以转发给SDN控制器。但是这样的处理方式存在许多问题，后来的driver被命名为xxx-networking的项目，主要是为了更好地对接HA部署的OpenStack，因为Neutron server分布在多个物理机上的时候会出现更加复杂的情况，而导致SDN控制器接受到错误的消息，这篇文章以ODL-Networking为例，介绍ODL是如何实现的，自研控制器的对接也可以参考ODL的设计思想。

ODL的ODL-Networking项目位于下面的链接中  
https://github.com/openstack/networking-odl  

## ODL-Networking 介绍


OpenStack networking-odl是一个二进制文件的一个plugin，它的作用是集成OpenStack的Neutron API和OpenDaylight SDN控制器。networking-odl有ML2 driver和L3 plugin的模块，可以支持OpenStack Neutron L2和L3的API，再转发数据到OpenDaylight控制器上

##   ODL驱动架构v1

ODL-Networking分为v1 driver和v2 driver
v1 driver会把OpenStack上的网络的更改同步到ODL控制器上。举个例子来说，用户创建了一个网络，首先这个操作会被写进OpenStack的数据库中，然后ODL driver会发送一个POST请求到ODL控制器  

虽然这个操作看起来很简单，但是却有一些隐含的问题 
 
* ODL并不是实时反应的，即使REST call成功给到了ODL,ODL也不一定可以马上响应  
* 在负载很大的时候（可能是odl或者openstack）,两边的同步可能会存在瓶颈  
* 在遇到odl与openstack同步失败的时候，v1 driver在下次的call REST API的时候会尝试去“full sync”整个neutron DB，所以会导致下次的REST API call花费很长的时间  
* 在多个rest api call竞争的情况下可能会出问题：  
1.例如，创建subnet的时候再创建port的情况，再neutron的HA部署的时候，这2个创建的消息会同步的通过v1 driver发给ODL,导致创建port失败  
2.在前面的“full sync”的操作的时候，这个时候数据库如果正在删除资源，那么这个时候同步的话，会把下一时刻会被删除的资源，同步到odl这边，形成openstack和odl的资源不同步


##    ODL驱动架构v2
面对前面的这些问题，社区又重新设计了v2 driver  

* v2 driver最重要的机制是引入了journaling（记录）机制，v2 driver会把openstack传给open daylight的数据记录在一个journal表中，journal表使用一堆队列来构成的。而不是直接把openstack的数据传递给odl  
* v2 driver中会起一个journal的线程，它会周期性地检查journal表中是否有新的数据，然后处理新的数据，另外的，openstack neutron的postcommmit的操作也会触发这个线程工作  
* 用户在创建一个网络资源的时候，这条数据会一开始存在neutron DB中，v2 driver再把这条数据存在“journal entry”里面，这个时候触发journal线程来处理这条数据  
* 数据在pre-commit的时候就已经被记录在journal entry中了，以防止这个commit失败的时候，journal entry也可以马上中止，实现消息的同步


##  Journal Entry架构
* journal entry一开始创建的时候，把状态置为“pending”等待状态，在这种状态下，entry在等待线程来处理，很多线程在处理entry的时候，只有一个线程可以“选中”这个entry
* 当这个entry被选中时，状态会被置为“processing”状态，并且被lock住，防止其他的线程来处理这个entry
* 当线程开始处理的时候，它先检查这个entry的依赖关系，如果这个entry依赖其他的entry，那么这个线程会把这个entry放回原位，然后再处理下一个entry
* 当这个entry没有其他依赖关系的时候，这个线程会吧entry里面的消息传递给opendaylight，具体是PUT,POST,DELETE这些操作，当这个REST call成功的时候，那么把这个entry的状态置为“completed”
* 当出现错误的时候，线程首先判断这个是“expected”意料之中的错误（如网络连通性等问题）还是“unexpected”意料之外的错误。对于那些意料之外的错误，会引入一个计数器，并对这个计数器设置一个数字，这个entry会被再次处理，处理的次数是这个计数器的数字。对于那些意料之中的错误，不会改变计数器的数字，当entry再次处理，直到计数器数字的次数后，这个entry就被设置为“failed”状态。否则，这个entry重新被标记为“pending”等待，等待下次被处理。


## Entry依赖检查机制
以下是社区的Entry依赖检查机制BP，Entry依赖检查机制将会被提前到entry创建的时候进行  
https://blueprints.launchpad.net/networking-odl/+spec/dep-validations-on-create  
当前v2 driver的Entry依赖检查机制发生在entry被处理的时候，但是将会被改为entry被创建的时候就进行依赖检查机制
当原先Entry依赖检查机制发生在entry被处理的时候，如果依赖检查机制fail，那么这个entry就会被送回队列，这样的处理方式可能会有以下几个问题
* entry依赖检查机制fail但是不知道哪里出了问题
* 很难去debug
* 当循环依赖的情况发生的话很难定位问题
* 一个entry可能会被反复进行依赖检查，导致CPU浪费
* 过多地检查依赖关系的话会导致数据库压力剧增

因此，Entry依赖检查机制将会被提前到entry创建的时候就进行，在entry被创建的时候，journal就会检查这个entry是否有依赖其他的entry，如果有的话，那么把这个entry放在一个链表中。这样的话，journal在选择entry的时候就会只去选择没有其他依赖的entry，当entry被处理后，无论是成功还是失败，依赖这个entry的链表都会被清空，这样可以使得后面的entry不会依赖到它。  
下面是journal依赖的表，一个entry如果有依赖的话，那么下面的表中的行列就是这个entry的外键，entry被删除的时候，这些外键也需要被同时删除。entry在被处理的时候，这个entry所依赖的parent被存入parent_id中，依赖这个entry的被存入dependent_id中，在journal处理这个entry的时候，必须要保证这个entry没有与parent有依赖关系（当parent被处理完之后会自动断开所有依赖），这样的话就保证了不会存在parent没处理完，而直接去处理这个entry的情况。  

```
+------------------------+
| odl_journal_dependency |
+------------------------+
| parent_id              |
| dependent_id           |
+------------------------+
```


## Entry恢复机制
以下是社区的Entry恢复机制BP，  
https://docs.openstack.org/developer/networking-odl/specs/completed/ocata/journal-recovery.html  
Journal entry在处理失败的情况，entry需要再被处理  
entry可能由于以下几个原因处理失败

* 在最大尝试次数条件下，一直处理失败
* ODL中的数据和OpenStack Neutron数据库不一致
* Neutron和ODL中的数据同步失败
目前失败的entry就一直会存在journal表中，这样会影响journal表的性能，大量的失败的entry存在journal表中的话对性能的影响还是很大的

现在需要解决的问题是如何处理entry中标记为fail的entry  
下面是社区BP的流程图  

```

+-----------------+
| For each entry  |
| in failed state |
+-------+---------+
        |
+-------v--------+
| Query resource |
| on ODL (REST)  |
+-----+-----+----+
      |     |                          +-----------+
   Resource |                          | Determine |
   exists   +--Resource doesn't exist--> operation |
      |                                | type      |
+-----v-----+                          +-----+-----+
| Determine |                                |
| operation |                                |
| type      |                                |
+-----+-----+                                |
      |              +------------+          |
      +--Create------> Mark entry <--Delete--+
      |              | completed  |          |
      |              +----------^-+       Create/
      |                         |         Update
      |                         |            |
      |          +------------+ |      +-----v-----+
      +--Delete--> Mark entry | |      | Determine |
      |          | pending    | |      | parent    |
      |          +---------^--+ |      | relation  |
      |                    |    |      +-----+-----+
+-----v------+             |    |            |
| Compare to +--Different--+    |            |
| resource   |                  |            |
| in DB      +--Same------------+            |
+------------+                               |
                                             |
+-------------------+                        |
| Create entry for  <-----Has no parent------+
| resource creation |                        |
+--------^----------+                  Has a parent
         |                                   |
         |                         +---------v-----+
         +------Parent exists------+ Query parent  |
                                   | on ODL (REST) |
                                   +---------+-----+
+------------------+                         |
| Create entry for <---Parent doesn't exist--+
| parent creation  |
+------------------+

```
entry中标记为fail状态的entry不能对后面进来的entry产生影响  
BP的实现可以分为两个部分，第一个部分，当我们检查到entry处于fail状态的时候，这个entry的动作可能是create或者update操作，如果这个资源再ODL中并没有存在、生效的话，那么我们就创建一个新的"create resource"这样的entry出来
其中，处理这个fail的entry是由一个专门的线程来处理，称为"maintenance thread"

## 代码分析
v2的driver相比以前只有一个python文件来说，代码量大了许多，主要内容在networking_odl目录下  
其中bgpvpn,fwaas,lbaas,qos,sfc这几个高级功能都是针对ODL的北向接口开发的，因此参考意义不大。主要的内容还是journal机制的实现以及l2和l3的实现  

```
L2_RESOURCES = {ODL_SG: ODL_SGS,
                ODL_SG_RULE: ODL_SG_RULES,
                ODL_NETWORK: ODL_NETWORKS,
                ODL_SUBNET: ODL_SUBNETS,
                ODL_PORT: ODL_PORTS}
L3_RESOURCES = {ODL_ROUTER: ODL_ROUTERS,
                ODL_FLOATINGIP: ODL_FLOATINGIPS}
```

在判断pending还是processing的时候，通过session.query去数据库中查找  

```
def check_for_pending_or_processing_ops(session, object_uuid, seqnum=None,
                                        operation=None):
    q = session.query(models.OpendaylightJournal).filter(
        or_(models.OpendaylightJournal.state == odl_const.PENDING,
            models.OpendaylightJournal.state == odl_const.PROCESSING),
        models.OpendaylightJournal.object_uuid == object_uuid)

    if seqnum is not None:
        q = q.filter(models.OpendaylightJournal.seqnum < seqnum)

    if operation:
        if isinstance(operation, (list, tuple)):
            q = q.filter(models.OpendaylightJournal.operation.in_(operation))
        else:
            q = q.filter(models.OpendaylightJournal.operation == operation)
    return session.query(q.exists()).scalar()
```

上述的2个机制分别位于2个类中，实现了描述的功能  
Journal Entry架构：class OpendaylightJournalThread(object):  
Entry恢复机制：class MaintenanceThread(object):  
而L2和L3	的转发分别位于以下2个类中：  

```
class OpenDaylightL2gwDriver(service_drivers.L2gwDriver):

class OpenDaylightL3RouterPlugin(
        common_db_mixin.CommonDbMixin,
        extraroute_db.ExtraRoute_db_mixin,
        l3_dvr_db.L3_NAT_with_dvr_db_mixin,
        l3_gwmode_db.L3_NAT_db_mixin,
        l3_agentschedulers_db.L3AgentSchedulerDbMixin):
```

在性能方面，ODL-Networking对对entry做cache缓存，有timeout和value属性，应该会有不小的性能提升  
class CacheEntry(collections.namedtuple('CacheEntry', ['timeout', 'values'])):  
为了更好的支持ODL增加的北向接口，ODL-Networking还特定会起一个start_odl_websocket_thread，位于下面的类中  
class OpendaylightWebsocketClient(object):  




##   参考文献
https://docs.openstack.org/developer/networking-odl/