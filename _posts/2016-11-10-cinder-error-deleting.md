---
layout: post
title: OpenStack删除Cinder盘失败解决办法
description: "OpenStack删除Cinder盘失败解决办法"
tags: [OpenStack]
categories: [OpenStack]
---


## 问题
Openstack Mitaka版本，终止了云主机之后，发现无法删除对应的云硬盘，删除提示报错为云硬盘的状态不是错误或者可用状态

![image](/images/openstack_cinder_error_deleting/1.png)

## 思路
切换至admin用户，进入数据库手动更新云硬盘的状态至错误状态


##  操作


查看云硬盘状态：  
```
# cinder list |grep error ```

![image](/images/openstack_cinder_error_deleting/2.png)


命令行删除，提示报错说还有依赖的快照。  


```
# cinder delete XXX
```

```
Delete for volume XXX failed: Invalid volume: Volume still has 1 dependent snapshots. (HTTP 400) (Request-ID: req-5ba025fb-5a61-422b-b00a-556e19083bd5)
ERROR: Unable to delete any of the specified volumes.
```

![image](/images/openstack_cinder_error_deleting/3.png)



方法有很多，这里介绍一种简单的。采取暴力手段，进入元数据库。  

![image](/images/openstack_cinder_error_deleting/4.png)


```
show databases;
```

![image](/images/openstack_cinder_error_deleting/5.png)


```
use cinder;
```


```
show tables;
```

![image](/images/openstack_cinder_error_deleting/6.png)

select找到出错的数据  

![image](/images/openstack_cinder_error_deleting/8.png)

删除元数据库中的数据,不过不能简单得把这个cinder盘的数据删除，以为数据库有外键依赖，而是要把cinder盘的error—deleting改成deleted

![image](/images/openstack_cinder_error_deleting/7.png)

再次查看云硬盘状态：  
![image](/images/openstack_cinder_error_deleting/9.png)

发现已经成功得删除了出错的cinder盘

总结：  
1、删除的时候注意id和volume-id两个字段，不要弄混掉了；
2、测试环境，暴力解决问题还是不太好，注意检查日志来对症下药。  
3、不要简单得去删除表中数据，而是需要更改状态