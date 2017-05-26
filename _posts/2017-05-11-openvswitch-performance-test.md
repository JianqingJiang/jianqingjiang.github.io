---
layout: post
title: OpenvSwitch Performance Test
description: "Openv Switch Performance Test"
tags: [SDN]
categories: [SDN]
---

##  实验环境：
首先介绍一下实验环境  
系统：CentOS7  
CPU:Intel(R) Xeon(R) CPU E5-2630 @ 2.30GHz  
Memory:DDR4 1600MHZ 16GB  
OVS版本：2.5.0  
拓扑描述：  
再两台物理服务器上搭建OpenStack计算节点，两台物理服务器之间通过INTEL 100G网卡进行连接，保证物理带宽够用。计算节点上分别启动5台虚拟机，计算节点上面启动着OVS组件，同时虚拟机连接到OVS。  
下图是部署在物理服务器上的2台OVS：
  
![image](/images/openvswitch_performance_test/1.png)
##  实验：
首先在OpenStack中每个物理服务器上起5台虚拟机，每台物理机上有5台，通过OVS进行连接，OVS又通过100G链路进行连接。

![image](/images/openvswitch_performance_test/2.png)

下图是物理的拓扑图，每个OVS下面挂了5台虚拟机，为了可以使得OVS可以满负荷运载  

![image](/images/openvswitch_performance_test/3.png)

再5对虚拟机上使用IPERF进行打流，即跨物理机的5对同时打流  

![image](/images/openvswitch_performance_test/4.png)

通过TOP命令观察物理服务器的资源消耗，下图得出资源消耗率比较低  

![image](/images/openvswitch_performance_test/5.png)

下图是在OVS上面下发的流表，可以使得2台OVS上面的5对虚拟机可以顺利通信  

![image](/images/openvswitch_performance_test/6.png)

以下就是打流的最终结果了，使用IFTOP命令观察物理服务器上面的所有流量，可以看到5对虚拟机分别的流量，以及最后的流量的总和，可以看到OVS的流量可以达到30G以上。  

![image](/images/openvswitch_performance_test/7.png)

总结：OpenvSwitch使用了2.5.0版本，可以看出在吞吐量上面已经可以轻松达到30G以上，这对于云计算的组网来说已经是比较高了，因为目前物理链路大部分还是10G的链路。


