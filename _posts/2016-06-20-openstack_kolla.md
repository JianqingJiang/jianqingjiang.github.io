---
layout: post
title: OpenStack Kolla实践
description: "OpenStack Kolla实践"
tags: [OpenStack]
categories: [OpenStack]
---

##    购买国外vps
由于kolla中的image太庞大了，超过10G。而且存放镜像的国外服务器会被block掉，之前试了使用vps搭建pptp的VPN，但是速度只有小几百k。网络出现波动还会断开连接...  
所以建议购买比较高性能的vps，在vps中下载好然后再拉回本地。之前购买了1G内存，20G硬盘的阿里北美vps,可惜由于性能原因，下载过程中导致了vps宕机。我查了半天错，以为是Kolla本身的问题。于是我购买了digitalocean的vps，选择2G内存，40G硬盘。可以按小时计费，这点比较好。   
![vps](/images/openstack_kolla/1.png)

vps的带宽和CPU使用情况  

![vps](/images/openstack_kolla/5.png)

#   build image

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
```

加入 EPEL 源


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
yum install python-devel
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
install_type = source
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



###  docker save image

由于Kolla的image数量太多，一个个保存太耗费时间，于是就写了个shell脚本


```
docker images > /root/kolla/images.txtawk '{print $1}' /root/kolla/images.txt > /root/kolla/images_cut.txtwhile read LINEdo#echo $LINEdocker save $LINE > /home/$LINE.tarecho okdone < /root/kolla/images_cut.txtecho finish
```


### 使用rsync拷贝到本地

```
CentOS/Fedora/RHEL: yum install rsync
vi /etc/rsyncd.conf
```

拷贝如下条目，path自己改  

```
motd file = /etc/rsyncd.motdlog file = /var/log/rsyncd.logpid file = /var/run/rsyncd.pidlock file = /var/run/rsync.lock[kolla]   path = /home/lokolla   comment = My Very Own Rsync Server   uid = nobody   gid = nobody   read only = no   list = yes   auth users = bnc   secrets file = /etc/rsyncd.scrt
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


```iptables -I INPUT -p tcp --dport 873 -j ACCEPTiptables -I OUTPUT -p tcp --sport 873 -j ACCEPT```
遇到报错

```rsync error: error starting client-server protocol (code 5) at main.c(1516) [Receiver=3.0.9]原因及解决办法：    SELinux；（下面这条命令在服务器端执行）    setsebool -P rsync_disable_trans on```
最后拷贝image到本地成功，速度也OK  
```
rsync -vzrtopg --progress -e ssh --delete root@45.55.240.159:/home/lokolla /home
```
![copy](/images/openstack_kolla/3.png)
###  docker load image

docker的镜像被拷贝回本地了。接下去就是把image load回来  

同样，也写了shell脚本  

```
#!/bin/sh#============ get the file name ===========Folder_A="/home/lokolla"for file_a in ${Folder_A}/*; do    temp_file=`basename $file_a`#   echo $temp_filedocker load < /home/lokolla/$temp_fileecho okdoneecho finish
```![image](/images/openstack_kolla/2.png)

然后docker images一下就发现镜像都OK了  


# run container


###   globals.yml
依然是先修改配置文件，与 Deploy 相关的主要是两个配置文件 /etc/kolla/passwords.yml 和 /etc/kolla/globals.yml。他们为 ansible 提供一些变量的设定。主要需要修改的是 globals.yml 文件。修改后，其主要内容为 :


```
config_strategy: "COPY_ALWAYS"
kolla_base_distro: "centos"
kolla_install_type: "source"
kolla_internal_address: "10.2.0.254"
network_interface: "eth0"
neutron_external_interface: "eth1"
openstack_logging_debug: "true"
enable_cinder: "no"
enable_heat: "no"
```


###   passwords.yml


这个密码文件可以使用工具自动生成，也可以手动输入  
但是手动输入需要注意格式，在：后需要空一格再输入；而且ssh_key也比较麻烦  
所以推荐使用工具自动生成  
但是直接输入  

```
kolla-genpwd
```
会提示找不到此命令。这也是官方文档坑的地方，改为：  

```
./root/kolla/.tox/genconfig/bin/kolla-genpwd
```



###  prechecks

kolla 使用一个名为 kolla-ansible 的封装脚本，可以使用 

```
./tools/kolla-ansible prechecks 
```
来检查一个机器是否满足安装条件。

precheck在其他地方都没什么问题,在IP上会进行下面两步检查，这个IP需要是node之间同一网段的，并且还需要不能被使用的IP地址  

```
TASK [prechecks : Checking if kolla_internal_vip_address and kolla_external_vip_address are not pingable from any node] 
TASK [prechecks : Checking if kolla_internal_vip_address is in the same network as network_interface on all nodes]
```

##  Deploy部署


使用 

```
./tools/kolla-ansible deploy
```

 来开始正式安装。
 

在deploy中，Kolla返回这个log，提示需要在外部的registry。但是all-in-one的安装并不需要registry，这个bother了我很久  
我在社区反馈了个bug  
[https://bugs.launchpad.net/kolla/+bug/1595128](https://bugs.launchpad.net/kolla/+bug/1595128 "https://bugs.launchpad.net/kolla/+bug/1595128")

```
Unknown error message: Tag 2.0.2 not found in repository docker.io/kollaglue/centos-source-heka
```
思路如下：那么我在本地自建一个registry，把下载好的image全部pull到本地的registry。然后把Kolla的外部registry指向我本地的registry  

奇怪的是，这个path貌似改不了，比如下载好的image tag是lokolla，这边查找的tag是kollaglue。即使我在/etc/kolla/kolla-build.conf中已经

```
[DEFAULT]
namespace = lokolla
```
这样编辑了

为了指向我的本地registry，在/etc/kolla/kolla-build.conf添加下面的条目

```
docker_registry: "localhost:4000"
openstack_release: "latest"
```
###  搭建本地registry

```
docker run -d -p 4000:5000 --restart=always --name registry registry:2
```
注意：kolla使用了4000端口   
Kolla looks for the Docker registry to use port 4000. (Docker default is port 5000)  
docker 暴露registry端口的方式  

```
EXPOSE (incoming ports)　　Dockefile在网络方面除了提供一个EXPOSE之外，没有提供其它选项。下面这些参数可以覆盖Dockefile的expose默认值：	--expose=[]: Expose a port or a range of ports from the container            without publishing it to your host	-P=false   : Publish all exposed ports to the host interfaces	-p=[]      : Publish a container᾿s port to the host (format:             ip:hostPort:containerPort | ip::containerPort |             hostPort:containerPort | containerPort)             (use 'docker port' to see the actual mapping)	--link=""  : Add link to another container (name:alias)    --expose可以让container接受外部传入的数据。container内监听的port不需要和外部host的port相同。比如说在container内部，一个HTTP服务监听在80端口，对应外部host的port就可能是49880.　　操作人员可以使用--expose，让新的container访问到这个container。具体有三个方式：　　1. 使用-p来启动container。　　2. 使用-P来启动container。　　3. 使用--link来启动container。　　如果使用-p或者-P，那么container会开发部分端口到host，只要对方可以连接到host，就可以连接到container内部。当使用-P时，docker会在host中随机从49153 和65535之间查找一个未被占用的端口绑定到container。你可以使用docker port来查找这个随机绑定端口。　　当你使用--link方式时，作为客户端的container可以通过私有网络形式访问到这个container。同时Docker会在客户端的container中设定一些环境变量来记录绑定的IP和PORT。
```

编辑 /etc/sysconfig/docker  

```
# CentOS
other_args="--insecure-registry 192.168.1.100:4000"
```

编辑 /etc/systemd/system/docker.service.d/kolla.conf


```
# CentOS
[Service]
EnvironmentFile=/etc/sysconfig/docker
# It's necessary to clear ExecStart before attempting to override it
# or systemd will complain that it is defined more than once.
ExecStart=
ExecStart=/usr/bin/docker daemon -H fd:// $other_args
```

重启docker  

```
# CentOS
systemctl daemon-reload
systemctl stop docker
systemctl start docker
```

在本地的registry如图所示  

![registry](/images/openstack_kolla/8.png)

停止registry的命令

```
docker stop registry && docker rm -v registry
```

写了shell脚本自动化把image push到docker仓库（registry）中，为了满足报错的log，并进行了tag的转换（lokolla->kollaglue）

```
docker images > /root/kolla/images.txtwhile read LINEdoecho $LINE > /root/kolla/meta.txtawk '{print $1}' /root/kolla/meta.txt > /root/kolla/images_1.txtawk '{print $3}' /root/kolla/meta.txt > /root/kolla/images_3.txtstr=$(cat /root/kolla/images_1.txt)echo /kollaglue/${str#*/} > /root/kolla/images_1_new.txtdocker tag $(cat /root/kolla/images_3.txt) localhost:4000$(cat /root/kolla/images_1_new.txt)echo tag okdocker push localhost:4000$(cat /root/kolla/images_1_new.txt)echo push ok#echo $(cat /root/kolla/images_3.txt)#echo $(cat /root/kolla/images_1.txt)#echo $(cat /root/kolla/images_1_new.txt)done < /root/kolla/images.txtecho finish
```

重新使用 ./tools/kolla-ansible deploy 来开始正式安装。  
使用 ./tools/kolla-ansible post-deploy 来生成 /etc/kolla/admin-openrc.sh 文件用来加载认证变量。
最后container全部up起来  
![registry](/images/openstack_kolla/7.png)
在浏览器上输入kolla_internal_address: "192.168.52.197"  域default，用户名密码在/etc/kolla/admin-openrc.sh中  登陆后的界面如下  
![registry](/images/openstack_kolla/9.png)这样就成功了，不过接下去还有很多“坑要踩”，趁着kolla项目代码量没上去还是值得好好研究一下的。容器化已经势不可挡了。

###  常见问题

* 今天Kolla镜像又下载不了了。以为是网络问题。进入openstack-kolla的irc聊天组。大牛基本都在这里聊天，发现有公告，原来是服务器正在重建。  


```
static.openstack.org (which hosts logs.openstack.org and tarballs.openstack.org among others) is currently being rebuilt. As jobs can not upload logs they are failing with POST_FAILURE. This should be resolved soon. Please do not recheck until then.
```



* 如果在deploy中出现这个问题。在/etc/hosts中加入"127.0.0.1  localhost"即可  

```
TASK: [rabbitmq | fail msg="Hostname has to resolve to IP address of api_interface"] ***failed: [localhost] => (item={'cmd': ['getent', 'ahostsv4', 'localhost'], 'end': '2016-06-24 04:51:39.738725', 'stderr': u'', 'stdout': '127.0.0.1       STREAM localhost\n127.0.0.1       DGRAM  \n127.0.0.1       RAW    \n127.0.0.1       STREAM \n127.0.0.1       DGRAM  \n127.0.0.1       RAW    ', 'changed': False, 'rc': 0, 'item': 'localhost', 'warnings': [], 'delta': '0:00:00.033351', 'invocation': {'module_name': u'command', 'module_complex_args': {}, 'module_args': u'getent ahostsv4 localhost'}, 'stdout_lines': ['127.0.0.1       STREAM localhost', '127.0.0.1       DGRAM  ', '127.0.0.1       RAW    ', '127.0.0.1       STREAM ', '127.0.0.1       DGRAM  ', '127.0.0.1       RAW    '], 'start': '2016-06-24 04:51:39.705374'}) => {"failed": true, "item": {"changed": false, "cmd": ["getent", "ahostsv4", "localhost"], "delta": "0:00:00.033351", "end": "2016-06-24 04:51:39.738725", "invocation": {"module_args": "getent ahostsv4 localhost", "module_complex_args": {}, "module_name": "command"}, "item": "localhost", "rc": 0, "start": "2016-06-24 04:51:39.705374", "stderr": "", "stdout": "127.0.0.1       STREAM localhost\n127.0.0.1       DGRAM  \n127.0.0.1       RAW    \n127.0.0.1       STREAM \n127.0.0.1       DGRAM  \n127.0.0.1       RAW    ", "stdout_lines": ["127.0.0.1       STREAM localhost", "127.0.0.1       DGRAM  ", "127.0.0.1       RAW    ", "127.0.0.1       STREAM ", "127.0.0.1       DGRAM  ", "127.0.0.1       RAW    "], "warnings": []}}msg: Hostname has to resolve to IP address of api_interfaceFATAL: all hosts have already failed -- abortingPLAY RECAP ********************************************************************           to retry, use: --limit @/root/site.retrylocalhost                  : ok=87   changed=24   unreachable=0    failed=1
```
  
* 有时端口号可能被占用也会在precheck中提示，通过端口号找到进程之后就可以用kill －9来杀死了  

```
yum install net-tools
netstat -apn|grep 5000(端口号)
```


### 工具
在/root/kolla的目录下有几个工具,可以在你deploy到一半中断的时候把本机环境清理一下
```
tools/cleanup-containers
tools/cleanup-host
#有时需要配合这两个命令：
docker kill $(docker ps -a -q) //杀死所有正在运行的containner
docker rm $(docker ps -a -q)   //删除已经被杀死的container
```  ###  版本变动

官网上说现在ansible版本>2.0也OK，但是我实际操作的时候首先precheck这里提示错误，deploy的时候也出错```
Some implemented distro versions of Ansible are too old to use distro packaging. Currently, CentOS and RHEL package Ansible >2.0 which is suitable for use with Kolla. Note that you will need to enable access to the EPEL repository to install via yum – to do so, take a look at Fedora’s EPEL docs and FAQ.
```

