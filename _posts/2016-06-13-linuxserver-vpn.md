---
layout: post
title: 利用vps搭建pptp vpn用于linux server翻墙
description: "利用vps搭建pptp vpn用于linux server翻墙"
tags: [linux]
categories: [linux]
---

###    VPS
可以购买AWS（使用visa信用卡可以免费一年）  
阿里云的北美服务器，或着digitalocean的vps  
我的环境：CentOS7（服务器端）

###    服务器端
使用一键安装脚本  
```
wget http://mirrors.linuxeye.com/scripts/vpn_centos.sh
chmod +x ./vpn_centos.sh
./vpn_centos.sh
```
脚本代码如下  
<pre><code>
#!/bin/bash
[ $(id -u) != "0" ] && { echo -e "\033[31mError: You must be root to run this script\033[0m"; exit 1; } 

export PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
clear
printf "
#######################################################################
#    LNMP/LAMP/LANMP for CentOS/RadHat 5+ Debian 6+ and Ubuntu 12+    #
#            Installs a PPTP VPN-only system for CentOS               #
# For more information please visit http://blog.linuxeye.com/31.html  #
#######################################################################
"

[ ! -e '/usr/bin/curl' ] && yum -y install curl

VPN_IP=`curl ipv4.icanhazip.com`

VPN_USER="linuxeye"
VPN_PASS="linuxeye"

VPN_LOCAL="192.168.0.150"
VPN_REMOTE="192.168.0.151-200"


while :; do echo
    read -p "Please input username: " VPN_USER 
    [ -n "$VPN_USER" ] && break
done

while :; do echo
    read -p "Please input password: " VPN_PASS
    [ -n "$VPN_PASS" ] && break
done
clear


if [ -f /etc/redhat-release -a -n "`grep ' 7\.' /etc/redhat-release`" ];then
    #CentOS_REL=7
    if [ ! -e /etc/yum.repos.d/epel.repo ];then
        cat > /etc/yum.repos.d/epel.repo << EOF
[epel]
name=Extra Packages for Enterprise Linux 7 - \$basearch
#baseurl=http://download.fedoraproject.org/pub/epel/7/\$basearch
mirrorlist=https://mirrors.fedoraproject.org/metalink?repo=epel-7&arch=\$basearch
failovermethod=priority
enabled=1
gpgcheck=0
EOF
    fi
    for Package in wget make openssl gcc-c++ ppp pptpd iptables iptables-services 
    do
        yum -y install $Package
    done
    echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
elif [ -f /etc/redhat-release -a -n "`grep ' 6\.' /etc/redhat-release`" ];then
    #CentOS_REL=6
    for Package in wget make openssl gcc-c++ iptables ppp 
    do
        yum -y install $Package
    done
    sed -i 's@net.ipv4.ip_forward.*@net.ipv4.ip_forward = 1@g' /etc/sysctl.conf
    rpm -Uvh http://poptop.sourceforge.net/yum/stable/rhel6/pptp-release-current.noarch.rpm
    yum -y install pptpd
else
    echo -e "\033[31mDoes not support this OS, Please contact the author! \033[0m"
    exit 1
fi

echo "1" > /proc/sys/net/ipv4/ip_forward

sysctl -p /etc/sysctl.conf

[ -z "`grep '^localip' /etc/pptpd.conf`" ] && echo "localip $VPN_LOCAL" >> /etc/pptpd.conf # Local IP address of your VPN server
[ -z "`grep '^remoteip' /etc/pptpd.conf`" ] && echo "remoteip $VPN_REMOTE" >> /etc/pptpd.conf # Scope for your home network

if [ -z "`grep '^ms-dns' /etc/ppp/options.pptpd`" ];then
     cat >> /etc/ppp/options.pptpd << EOF
ms-dns 223.5.5.5 # Aliyun DNS Primary
ms-dns 114.114.114.114 # 114 DNS Primary
ms-dns 8.8.8.8 # Google DNS Primary
ms-dns 209.244.0.3 # Level3 Primary
ms-dns 208.67.222.222 # OpenDNS Primary
EOF
fi

echo "$VPN_USER pptpd $VPN_PASS *" >> /etc/ppp/chap-secrets

ETH=`route | grep default | awk '{print $NF}'`
[ -z "`grep '1723 -j ACCEPT' /etc/sysconfig/iptables`" ] && iptables -I INPUT 4 -p tcp -m state --state NEW -m tcp --dport 1723 -j ACCEPT
[ -z "`grep 'gre -j ACCEPT' /etc/sysconfig/iptables`" ] && iptables -I INPUT 5 -p gre -j ACCEPT 
iptables -t nat -A POSTROUTING -o $ETH -j MASQUERADE
iptables -I FORWARD -p tcp --syn -i ppp+ -j TCPMSS --set-mss 1356
service iptables save
sed -i 's@^-A INPUT -j REJECT --reject-with icmp-host-prohibited@#-A INPUT -j REJECT --reject-with icmp-host-prohibited@' /etc/sysconfig/iptables 
sed -i 's@^-A FORWARD -j REJECT --reject-with icmp-host-prohibited@#-A FORWARD -j REJECT --reject-with icmp-host-prohibited@' /etc/sysconfig/iptables 
service iptables restart
chkconfig iptables on

service pptpd restart
chkconfig pptpd on
clear

echo -e "You can now connect to your VPN via your external IP \033[32m${VPN_IP}\033[0m"

echo -e "Username: \033[32m${VPN_USER}\033[0m"
echo -e "Password: \033[32m${VPN_PASS}\033[0m"

</code></pre>
执行结束后即安装完成  
###           客户端  

<pre><code>
yum install pptp  
</code></pre>
<pre><code>
modprobe nf_conntrack_pptp  
</code></pre>
<pre><code>
echo '用户名 PPTP 密码 *' >> /etc/ppp/chap-secrets   
</code></pre>

在/etc/ppp/peers/目录下新疆一个文本（本文为linuxconfig）  
    
<pre><code>
pty "pptp 123.123.1.1(vps ip) --nolaunchpppd"
name 用户名
password 密码
remotename PPTP
require-mppe-128
file /etc/ppp/options.pptp
ipparam linuxconfig
</code></pre>

连接VPN  

<pre><code>
pppd call linuxconfig
</code></pre>
断开VPN  

<pre><code>
pkill pppd
</code></pre>
PS:  
log在/var/log/messages目录下  
可以看到ip a显示下多了一个ppp0接口（如果这个接口时down的则重复尝试断开和连接步骤）  
###                        配置路由  
<pre><code>
ip route add default dev ppp0
</code></pre>
这样所有流量全部由VPN转发  
可以使用curl www.google.com测试VPN是否生效  

![curl](/images/vpn/vpn.png)

这样就可以配置完成了  
还有一些小trick就是注意vps上的DNS和镜像的源，厂商一般就更改为自己的，注意替换
这样一般是OK了，但是网速仍然才小几百k...科学上网技术仍有待提高  
 
![smile](/images/vpn/smile.gif)