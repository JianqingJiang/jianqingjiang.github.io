---
layout: post
title: OpenStack spice协议配置
description: "OpenStack spice协议配置"
tags: [OpenStack]
categories: [OpenStack]
---
## Enable SPICE HTML5 Console Access in OpenStack Mikata
环境: CentOS7 + OpenStack Mikata 
###  SPICE VS VNC  | BIOS屏幕显示        | 能           | 能|
|:--------|:-------:|--------:|
| 全彩支持      | 能 | 能 |
| 更改分辨率      | 能     |   能 |
| 多显示器 | 多显示器支持（高达4画面）      |    只有一个屏幕|
| 图像传输 | 图像和图形传输      |    图像传输 |
| 视频播放支持 | GPU加速支持      |    不能|
| 音频传输 | 双向语音可以控制      |   不能|
| 鼠标控制 | 客户端服务器都可以控制  |    服务器端控制 |
| USB传输 | USB可以通过网络传输    |    不能|
| 加密 | 通讯可以使用SSL进行加密      |    不能 |
{: rules="groups"}


###   Spice协议通信拓扑

![1](/images/openstack_spice/4.png) 


### Required Packages


On both Control & Compute:

```
yum install spice-html5
```

####  注意点

spice-html5 在epel源里，需要配置epel源

```
[epel]
name=Extra Packages for Enterprise Linux 7 - $basearch
#baseurl=http://download.fedoraproject.org/pub/epel/7/$basearch
mirrorlist=https://mirrors.fedoraproject.org/metalink?repo=epel-7&arch=$basearch
failovermethod=priority
enabled=1
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
```

On Control:

```
yum install openstack-nova-spicehtml5proxy
```


###  Config Files

The file to modify is

/etc/nova/nova.conf  
in compute and control nodes.  


In both config files, ensure  vnc_enabled=False is explicitly set. If novnc is enabled, ensure that is disabled too.(在这里把enabled=False即可)  


Control IP = 192.168.1.100  
Compute IP = 172.16.1.100   [Internal IP - ports may need to be opened if not already there]  

####   On Control Node

{% highlight css %}

/etc/nova/nova.conf
[DEFAULT]
web=/usr/share/spice-html5
. . .
[spice]

html5proxy_host=0.0.0.0
html5proxy_port=6082
html5proxy_base_url=https://192.168.1.100:6082/spice_auto.html

# Enable spice related features (boolean value)
enabled=True
 
# Enable spice guest agent support (boolean value)
agent_enabled=true
 
# Keymap for spice (string value)
keymap=en-us

{% endhighlight %}


#####   设置iptables

```
iptables -I INPUT -p tcp -m multiport --dports 6082 -m comment --comment "Allow SPICE connections for console access " -j ACCEPT
```

#####   永久设置iptables
You can make permanent by adding the above rule to /etc/sysconfig/iptables (before the reject rules) saving and restarting iptables.

####   Config Changes on Compute Node

{% highlight css %}

/etc/nova/nova.conf
[DEFAULT]
web=/usr/share/spice-html5
. . .
[spice]

html5proxy_base_url=https://192.168.1.100:6082/spice_auto.html
server_listen=0.0.0.0
server_proxyclient_address=172.16.10.100

# Enable spice related features (boolean value)
enabled=True
 
# Enable spice guest agent support (boolean value)
agent_enabled=true
 
# Keymap for spice (string value)
keymap=en-us

{% endhighlight %}



###  Restart services


####   On Compute

```
# service openstack-nova-compute restart
```

####   On Control

```
# service httpd restart
# service openstack-nova-spicehtml5proxy start
# service openstack-nova-spicehtml5proxy status 
# systemctl enable openstack-nova-spicehtml5proxy
```

在Control node上看到6082端口在监听

![2](/images/openstack_spice/3.png) 


虚拟机需要重启才能使用spice协议


![3](/images/openstack_spice/2.png) 



OpenStack中的windows7播放视频，有点卡，由于在服务器中图像处理都是CPU来做的，需要优化spice协议

![4](/images/openstack_spice/1.jpg)