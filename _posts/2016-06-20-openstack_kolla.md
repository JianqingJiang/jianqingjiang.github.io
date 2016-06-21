---
layout: post
title: OpenStack Kolla实践
description: "OpenStack Kolla实践"
tags: [OpenStack]
categories: [OpenStack]
---

###    购买国外vps
由于kolla中的image太庞大了，超过10G。而且存放镜像的国外服务器会被block掉，之前试了使用vps搭建pptp的VPN，但是速度只有小几百k。网络出现波动还会断开连接...  
所以建议购买比较高性能的vps，在vps中下载好然后再拉回本地。之前购买了1G内存，20G硬盘的阿里北美vps,可惜由于性能原因，下载过程中导致了vps宕机。我查了半天错，以为是Kolla本事的问题。于是我购买了digitalocean的vps，选择2G内存，40G硬盘。可以按小时计费，这点比较好。   
![vps](/images/openstack_kolla/1.png)



###   开始部署

Kolla 依赖于以下几个主要组件

* Ansible > 1.9.4, < 2.0
* Docker > 1.10.0
* docker-py > 1.7.0
* python jinja2 > 2.6.0

几点说明：

* 机器使用的是vmware虚拟机进行的测试。配置上使用 16G RAM, 4 CPU, 2 网卡的配置
* 由于使用了 Docker，所以对于底层系统并还没什么要求，本文使用 CentOS 7 系统。
* Kolla master 分支上使用的是 RDO master 上的源，打包极不稳定，时常会有 Bug 出现。所以本文使用的是 CentOS + 源码的安装方式

###  加入 Docker 的源
```
	sudo tee /etc/yum.repos.d/docker.repo << 'EOF'
	[dockerrepo]
	name=Docker Repository
	baseurl=https://yum.dockerproject.org/repo/main/centos/$releasever/
	enabled=1
	gpgcheck=1
	gpgkey=https://yum.dockerproject.org/gpg
	EOF
	加入 EPEL 源
```


```
yum install -y epel-release
```

###   安装 Kolla 所需依赖

```
yum install -y ansible docker-engine git gcc python-setuptools
easy_install -U pip
```
Docker 现在使用了 shared mount 功能，默认没有打开，需要手动修改 Docker 启动文件 /usr/lib/systemd/system/docker.service 的 MountFlags

```
sed -i 's/MountFlags.*/MountFlags=shared/' /usr/lib/systemd/system/docker.service
```

###   启动 Docker 服务

```
systemctl daemon-reload
systemctl enable docker
systemctl start docker
```



###  下载 Kolla 代码并安装依赖

```
git clone https://github.com/openstack/kolla.git
cd kolla
pip install -r requirements.txt -r test-requirements.txt tox
```



###  Build Docker Image

以下如果没有特别说明，所有的操作都是在 Kolla 项目的目录里进行
首先要先生成并修改配置文件

```
tox -e genconfig
cp -rv etc/kolla /etc/
```
然后修改 /etc/kolla/kolla-build.conf 文件，它是用来控制 kolla build 过程的。修改后，其主要内容如下 :


```
[DEFAULT]
base = centos
install_type = soruce
namespace = lokolla
push = false
```

目前使用mitaka 分支，master分支不稳定  

```
git checkout origin/stable/mitaka
```
接下来就是进行漫长的 build, 这个过程主要依赖机器性能和网速。如果快的话，20 多分钟就完成。如果有问题的话，会很久。不过依赖于 Docker Build 的 Cache 功能，就算重跑的话，之前已经 Build 好的也会很快完成。

```
./tool/build.py -p default
```
参数中的 -p default 是指定了只 build 主要的 image, 包括 : mariadb, rabbitmq, cinder, ceilometer, glance, heat, horizon, keystone, neutron, nova, swift 等 . 这些可以只生成的 kolla-build.conf 里找到。


所有image全部下载完成之后如图  

![iamge](/images/openstack_kolla/2.png)

### 使用rsync拷贝到本地

```
CentOS/Fedora/RHEL: yum install rsync
vi /etc/rsyncd.conf
```

拷贝如下条目，path自己改  

```
motd file = /etc/rsyncd.motd
```

编辑下面文件  

```
vi /etc/rsyncd.secrets
```
输入密码  

```
bnc:bnc
```
/etc/rsyncd.secrets 文件权限必须是600，创建好该文件后可以执行： chmod 600 /etc/rsyncd.secrets  

如果开启了iptables防火墙，请将873端口加入防火墙允许规则


```


```



###  常见问题

今天Kolla镜像又下载不了了。以为是网络问题。进入openstack-kolla的irc聊天组。大牛基本都在这里聊天，发现有公告  

```
static.openstack.org (which hosts logs.openstack.org and tarballs.openstack.org among others) is currently being rebuilt. As jobs can not upload logs they are failing with POST_FAILURE. This should be resolved soon. Please do not recheck until then.
```

原来是服务器正在重建。  