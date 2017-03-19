---
layout: post
title: OpenvSwitch Dpdk大页配置
description: "OpenvSwitch Dpdk大页配置"
tags: [OpenvSwitch]
categories: [OpenvSwitch]
---

## grub配置
在/etc/grub2.cfg里改参数  
需要在/etc/default/grub里面加上 GRUB_CMDLINE_LINUX_DEFAULT="default_hugepagesz=1GB hugepagesz=1G hugepages=4 iommu=pt intel_iommu=on"，如果/etc/default/grub不存在，则创建一个，加上这些kernel参数。  

然后再用update-grub，这些参数在/boot/grub2/grub.cfg里面不是在最后，而是在中间给vmlinuz的参数，比如像如下的例子：
linux   /vmlinuz-4.4.0-66-generic root=/dev/mapper/m4--u1604--vg-root ro  iommu=pt intel_iommu=on default_hugepagesz=1GB hugepagesz=1G hugepages=8



## 实验：
### 一、
1.没改/boot/grub2/grub.cfg里面加上iommu=pt intel_iommu=on default_hugepagesz=1GB hugepagesz=1G hugepages=8
cat /proc/cmdline显示如下：  
![image](/images/openvswitch-dpdk/1.jpg)

2.hugepage的显示如下
![image](/images/openvswitch-dpdk/2.jpg)

3.hugepage的该路径里面只有2048K这个目录
![image](/images/openvswitch-dpdk/3.jpg)


###   二、
1.在/boot/grub2/grub.cfg里面相关条目改成如下的设置，然后重启机器
![image](/images/openvswitch-dpdk/4.jpg)

2.cat /proc/cmdline显示

![image](/images/openvswitch-dpdk/5.jpg)

3.hugepage的显示如下，只有4个1G了

![image](/images/openvswitch-dpdk/6.jpg)

4.hugepage的该路径里面只有1048576K这个目录

![image](/images/openvswitch-dpdk/7.jpg)

1G大页和2M大页根据物理环境和负载不同会有不同的影响，2M大概会有10-20%左右的性能降低。
另外用了1G大页，就不用设置sysctl 的vm.nr_hugepages参数了。


##  100G网卡打流试验
1.使用2张100G的网卡，在物理机os上进行iperf打流，实验结果差不多为46G左右，达到了5成左右
2.然后在100G的网卡连接的2台机器上分别安装mininet，通过创建分布在两个物理机上的虚拟机进行iperf打流，流量经过openvswitch，实验结果差不多为43G左右。