---
layout: post
title: OpenStack Boston Summit直击
description: "OpenStack Boston Summit直击"
tags: [OpenStack]
categories: [OpenStack]
---

## OpenStack Boston Summit Day1 Keynote

2017年波士顿OpenStack Summit在Hynes会议中心如期举办

![image](/images/openstack_boston_summit/1.png)

Keynote会场

![image](/images/openstack_boston_summit/2.png)

下面是本次峰会的主要赞助商

![image](/images/openstack_boston_summit/3.png)

## Keynote简述

这次的峰会主题重点从周一上午的Keynote中就可以看出，这次的Keynote一开始就强调了私有云（Private Cloud）的重要性，以及公有云和私有云之间的对（si）比（bi）。不像之前大家都在讨论公有云那样，这段时间大家都开始把自己的业务迁移到了自家建的私有云中。Keynote的后半部分强调了容器和谷歌的K8S的技术前景和Ebay在容器技术方面的实践，也阐述了今后技术演化的路线。  

Keyword: Private Cloud、OpenCloud、Containers  
接下去是各家厂商及用户的一些分享：  
![image](/images/openstack_boston_summit/4.png)
## GE
然后GE的人就上来强调了GE有多少业务迁移到了自家的私有云中  
接着给出了以下的一些数据  
530 applications migrated to cloud  
42% applications in cloud  
Shared automation strategy across public&private  
![image](/images/openstack_boston_summit/5.png)
## US Army
US Army强调了OpenStack在减少开支方面的主要作用，OpenStack在 他们的教育系统“Education with agile development”方面减少了很多开支，而且使得教育方式变得十分灵活高效。而且把业务迁移到了OpenStack之后可以帮助他们确实的减少开支  
![image](/images/openstack_boston_summit/6.png)
## Mirantis
接下去是mirantis公司说出了一些担忧，public cloud把大家的注意力都吸引走了，但是真正的要建设一个opencloud的大环境，需要private cloud的成功。AWS的提及，以及要建立一个真正的opencloud是由public cloud垂直的自顶向下的设计，而不是一个企业软件的缝缝补补。  
![image](/images/openstack_boston_summit/7.png)
## AT&T
接下去是AT&T的director的topic，他们的重点是“Next generation video platform,AT&T”他们也会更加关注更多业务层方面的东西，当然他们也在使用OpenStack作为重要的IaaS层。
 
## Ebay 
Ebay首先说了他们对基础设施的要求，因为他们的业务量也很大,他们引入了Docker，Kubernetes等新的技术。首先他们的k8s是powered by openstack，他们喜欢opensource  
“Managing k8s at scale”是他们的目前主要工作。  
![image](/images/openstack_boston_summit/8.png)
## Redhat
Redhat是openstack峰会的元老，先是强调了一下openstack以及取得的巨大成果，以及介绍了redhat的openstack用户，和帮助这些用户们取得了巨大的进步， “innovation evolution”
和“openstack+containers”已经成为今后的技术方向。  

基本上从第一天的Keynote上就可以看出整个OpenStack社区的关注点和技术发展的趋势，以及用户对开源社区提出了哪些新的需求。无疑，容器和Kubernetes正在成为新宠儿。  

## OpenStack Boston Summit Day2 Keynote

第二天的Keynote由OpenStack的CO-Founder Mark Collier来主持，第二天简直就是Kuberneters的Keynote了，上来的演示几乎都要展示一个Kuberneters的界面。而提到架构也是OpenStack上面就是Kuberneters了，见下图。  
![image](/images/openstack_boston_summit/9.png)
Ironic的一个演示，由IBM来进行演示，先是通过Baremetal来启动各种OpenStack服务。其中多次提到了编排层通过Kuberneters来进行编排服务。由OpenStack来提供IaaS层，再由Kuberneters来进行对容器的编排。
![image](/images/openstack_boston_summit/10.png)
接下去是mirantis的演示，是一个Bigdata analydemo通过容器的形式，由Kuberneters来进行编排。也是通过Baremetal的环境来进行演示。  

Intel推了他们的“clear containers”，也是准备在容器的层面发力。他们也强调了他们的芯片设计，如何对OpenStack有更好的贡献，Intel还强调了与华为的一个BugSmash的合作，在国内也是颇有影响力。  
![image](/images/openstack_boston_summit/11.png)
Google Cloud CTO进行的分享是“开源”的话题，谷歌也在变得越来越开源，还提到了AI技术，几年前觉得AI是20年以后的事，但是科技的发展已经变得越来越迅速。机器学习可以解决问题，而且也是开源的，非常的有价值。  
![image](/images/openstack_boston_summit/12.png)
COREOS说了一个cloud native的数据库 CockroachDB,这个数据库主要是为了解决mysql等传统数据库的扩展性，他们把CockroachDB部署在了多个Kuberneters的node上面，然后把一个node上面的CockroachDB关闭然后重启，再观察这个cluster的功能，其中CockroachDB在Kuberneters上面还有realtime的界面，可以对数据库的问题有更好的定位。  
![image](/images/openstack_boston_summit/13.png)
Open Telecom Cloud强调了混合云的需求  
Clould business will go hybrid  
Openstack has consistent success private cloud  
并强调了要使得混合云成功，需要各家厂商使用相同的接口  
![image](/images/openstack_boston_summit/14.png)
第二天最大的惊喜莫过于请来了斯诺登来讨论安全了，当斯诺登提到当用户把虚拟机和数据放在AWS或者其他的一些公有云上，怎么能够确定他们没有在监视你呢？还说了一写关于安全的更深层，更人性的一些问题，这里就不具体展开了。不过可以深切的感受到OpenStack对于开源，对于科技，对于人文的精神的理解。  
![image](/images/openstack_boston_summit/15.png)  


希望下次可以温哥华见
