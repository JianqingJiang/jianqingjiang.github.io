---
layout: post
title: OpenStack Magnum是如何在DevStack中启动的
description: "OpenStack Magnum是如何在DevStack中启动的"
tags: [OpenStack]
categories: [OpenStack]
---
##什么是magnum?
Mangum现在应该是OpenStack里边比较热门的一个和Docker集成的新项目。Magnum是去年巴黎峰会后开始的一个新的专门针对Container的一个新项目，用来向用户提供容器服务。从去年11月份开始在stackforge提交第一个 patch，今年3月份进入OpenStack namespace，这个项目应该是OpenStack社区从stackforge迁移到OpenStack namespace最快的一个项目。Magnum现在可以为用户提供Kubernetes as a Service、Swarm as a Service和这几个平台集成的主要目的是能让用户可以很方便的通过OpenStack云平台来管理k8s，swarm，这些已经很成型的Docker集群管理系统，使用户很方便的使用这些容器管理系统来提供容器服务。  
##使用devstack安装magnum
magnum依赖于nova,glance,heat,barbican,neutron这些组件来模拟一个物理的环境，在裸机上部署magnum社区还在开发中  
推荐使用Ubuntu14.04(Trusty)和Fedora 20/21  
首先 Clone devstack  
<pre><code>
cd ~
git clone https://git.openstack.org/openstack-dev/devstack
</code></pre>
配置devstack，enable heat和neutron  
<pre><code>
cd devstack
cat > local.conf << END
[[local|localrc]]
# Modify to your environment
FLOATING_RANGE=192.168.1.224/27
PUBLIC_NETWORK_GATEWAY=192.168.1.225
PUBLIC_INTERFACE=em1

# Credentials
ADMIN_PASSWORD=password
DATABASE_PASSWORD=password
RABBIT_PASSWORD=password
SERVICE_PASSWORD=password
SERVICE_TOKEN=password

enable_service rabbit

# Ensure we are using neutron networking rather than nova networking
# (Neutron is enabled by default since Kilo)
disable_service n-net
enable_service q-svc
enable_service q-agt
enable_service q-dhcp
enable_service q-l3
enable_service q-meta
enable_service neutron

# Enable heat services
enable_service h-eng
enable_service h-api
enable_service h-api-cfn
enable_service h-api-cw

# Enable barbican services
enable_plugin barbican https://git.openstack.org/openstack/barbican

FIXED_RANGE=10.0.0.0/24

Q_USE_SECGROUP=True
ENABLE_TENANT_VLANS=True
TENANT_VLAN_RANGE=

PHYSICAL_NETWORK=public
OVS_PHYSICAL_BRIDGE=br-ex

# Log all output to files
LOGFILE=$HOME/logs/devstack.log
SCREEN_LOGDIR=$HOME/logs

VOLUME_BACKING_FILE_SIZE=20G
END
</code></pre> 
创建local.sh，使的magnum能够使用devstack创建的网络  
<pre><code>
cat > local.sh << 'END_LOCAL_SH'
#!/bin/sh
ROUTE_TO_INTERNET=$(ip route get 8.8.8.8)
OBOUND_DEV=$(echo ${ROUTE_TO_INTERNET#*dev} | awk '{print $1}')
sudo iptables -t nat -A POSTROUTING -o $OBOUND_DEV -j MASQUERADE
END_LOCAL_SH
chmod 755 local.sh
</code></pre>
运行devstack  
<pre><code>
./stack.sh
</code></pre>
source环境变量  
<pre><code>
source ~/devstack/openrc admin admin
</code></pre>
把Fedora Atomic micro-OS存在glance中  
<pre><code>
cd ~
wget https://fedorapeople.org/groups/magnum/fedora-21-atomic-5.qcow2
glance image-create --name fedora-21-atomic-5 \
                    --visibility public \
                    --disk-format qcow2 \
                    --os-distro fedora-atomic \
                    --container-format bare < fedora-21-atomic-5.qcow2

</code></pre>
创建keypair来使用baymodel  
<pre><code>
test -f ~/.ssh/id_rsa.pub || ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
nova keypair-add --pub-key ~/.ssh/id_rsa.pub testkey
</code></pre>
为magnum创建MySql数据库  
<pre><code> 
mysql -h 127.0.0.1 -u root -ppassword mysql <<EOF
CREATE DATABASE IF NOT EXISTS magnum DEFAULT CHARACTER SET utf8;
GRANT ALL PRIVILEGES ON magnum.* TO
    'root'@'%' IDENTIFIED BY 'password'
EOF
</code></pre>
安装magnum  
<pre><code>
cd ~
git clone https://git.openstack.org/openstack/magnum
cd magnum
sudo pip install -e .
</code></pre>
配置magnum  
<pre><code>
# create the magnum conf directory
sudo mkdir -p /etc/magnum
# copy sample config and modify it as necessary
sudo cp etc/magnum/magnum.conf.sample /etc/magnum/magnum.conf
# copy policy.json
sudo cp etc/magnum/policy.json /etc/magnum/policy.json
# enable debugging output
sudo sed -i "s/#debug\s*=.*/debug=true/" /etc/magnum/magnum.conf
# set RabbitMQ userid
sudo sed -i "s/#rabbit_userid\s*=.*/rabbit_userid=stackrabbit/" \
         /etc/magnum/magnum.conf
# set RabbitMQ password
sudo sed -i "s/#rabbit_password\s*=.*/rabbit_password=password/" \
         /etc/magnum/magnum.conf
# set SQLAlchemy connection string to connect to MySQL
sudo sed -i "s/#connection\s*=.*/connection=mysql:\/\/root:password@localhost\/magnum/" \
         /etc/magnum/magnum.conf
# set Keystone account username
sudo sed -i "s/#admin_user\s*=.*/admin_user=admin/" \
         /etc/magnum/magnum.conf
# set Keystone account password
sudo sed -i "s/#admin_password\s*=.*/admin_password=password/" \
         /etc/magnum/magnum.conf
# set admin Identity API endpoint
sudo sed -i "s/#identity_uri\s*=.*/identity_uri=http:\/\/127.0.0.1:35357/" \
         /etc/magnum/magnum.conf
# set public Identity API endpoint
sudo sed -i "s/#auth_uri\s*=.*/auth_uri=http:\/\/127.0.0.1:5000\/v2.0/" \
         /etc/magnum/magnum.conf
# set oslo messaging notifications driver (if using ceilometer)
sudo sed -i "s/#driver\s*=.*/driver=messaging/" \
         /etc/magnum/magnum.conf
</code></pre>
安装magnum客户端  
<pre><code>
cd ~
git clone https://git.openstack.org/openstack/python-magnumclient
cd python-magnumclient
sudo pip install -e .
</code></pre>
为magnum配置数据库  
<pre><code>
magnum-db-manage upgrade
</code></pre>
配置keystone的endpoint  
<pre><code>
openstack service create --name=magnum \
                          --description="Magnum Container Service" \
                          container
openstack endpoint create --region=RegionOne \
                          --publicurl=http://127.0.0.1:9511/v1 \
                          --internalurl=http://127.0.0.1:9511/v1 \
                          --adminurl=http://127.0.0.1:9511/v1 \
                          magnum
</code></pre>
启动magnum  
<pre><code>
magnum-api
magnum-conductor
</code></pre>
##Magnum关于DevStack启动的代码解读  
代码结构如下  
├── devstack  
│   ├── lib  
│   │   └── magnum  
│   ├── plugin.sh  
│   ├── README.rst  
│   ├── settings  
magnum中定义了magnum所创建文件的路径以及git镜像时的路径
<pre><code>
MAGNUM_REPO=${MAGNUM_REPO:-${GIT_BASE}/openstack/magnum.git}
MAGNUM_BRANCH=${MAGNUM_BRANCH:-master}
MAGNUM_DIR=$DEST/magnum

GITREPO["python-magnumclient"]=${MAGNUMCLIENT_REPO:-${GIT_BASE}/openstack/python-magnumclient.git}
GITBRANCH["python-magnumclient"]=${MAGNUMCLIENT_BRANCH:-master}
GITDIR["python-magnumclient"]=$DEST/python-magnumclient
MAGNUM_STATE_PATH=${MAGNUM_STATE_PATH:=$DATA_DIR/magnum}

MAGNUM_AUTH_CACHE_DIR=${MAGNUM_AUTH_CACHE_DIR:-/var/cache/magnum}
MAGNUM_CONF_DIR=/etc/magnum
MAGNUM_CONF=$MAGNUM_CONF_DIR/magnum.conf
MAGNUM_POLICY_JSON=$MAGNUM_CONF_DIR/policy.json
MAGNUM_API_PASTE=$MAGNUM_CONF_DIR/api-paste.ini
</code></pre>
定义好路径之后就创建各种配置文件。并进行检查。如果不存在则创建该文件，并赋予权限  
<pre><code>
function configure_magnum {
    # Put config files in ``/etc/magnum`` for everyone to find
    if [[ ! -d $MAGNUM_CONF_DIR ]]; then
        sudo mkdir -p $MAGNUM_CONF_DIR
        sudo chown $STACK_USER $MAGNUM_CONF_DIR
    fi
</code></pre>
由于magnum的认证需要依赖keystone。那么需要对mysql进行操作。需要创建服务并返回endpoint  
<pre><code>
function create_magnum_accounts {

    create_service_user "magnum" "admin"

    if [[ "$KEYSTONE_CATALOG_BACKEND" = 'sql' ]]; then

        local magnum_service=$(get_or_create_service "magnum" \
            "container" "Magnum Container Service")
        get_or_create_endpoint $magnum_service \
            "$REGION_NAME" \
            "$MAGNUM_SERVICE_PROTOCOL://$MAGNUM_SERVICE_HOST:$MAGNUM_SERVICE_PORT/v1" \
            "$MAGNUM_SERVICE_PROTOCOL://$MAGNUM_SERVICE_HOST:$MAGNUM_SERVICE_PORT/v1" \
            "$MAGNUM_SERVICE_PROTOCOL://$MAGNUM_SERVICE_HOST:$MAGNUM_SERVICE_PORT/v1"
    fi

}
</code></pre>
然后使用类似于shell的文件写入命令进行配置文件的写入操作  
<pre><code>
function create_magnum_conf {

    # (Re)create ``magnum.conf``
    rm -f $MAGNUM_CONF
    iniset $MAGNUM_CONF DEFAULT debug "$ENABLE_DEBUG_LOG_LEVEL"
    iniset $MAGNUM_CONF oslo_messaging_rabbit rabbit_userid $RABBIT_USERID
    iniset $MAGNUM_CONF oslo_messaging_rabbit rabbit_password $RABBIT_PASSWORD
    iniset $MAGNUM_CONF oslo_messaging_rabbit rabbit_host $RABBIT_HOST

    iniset $MAGNUM_CONF database connection `database_connection_url magnum`
    iniset $MAGNUM_CONF api host "$MAGNUM_SERVICE_HOST"
    iniset $MAGNUM_CONF api port "$MAGNUM_SERVICE_PORT"
</code></pre>
magnum可以选择多个底层OS  
<pre><code>
function magnum_register_image {
    local magnum_image_property="--property os_distro="

    local atomic="$(echo $MAGNUM_GUEST_IMAGE_URL | grep -io 'atomic' || true;)"
    if [ ! -z "$atomic" ]; then
        magnum_image_property=$magnum_image_property"fedora-atomic"
    fi
    local ubuntu="$(echo $MAGNUM_GUEST_IMAGE_URL | grep -io "ubuntu" || true;)"
    if [ ! -z "$ubuntu" ]; then
        magnum_image_property=$magnum_image_property"ubuntu"
    fi
    local coreos="$(echo $MAGNUM_GUEST_IMAGE_URL | grep -io "coreos" || true;)"
    if [ ! -z "$coreos" ]; then
        magnum_image_property=$magnum_image_property"coreos"
    fi

    openstack --os-url $GLANCE_SERVICE_PROTOCOL://$GLANCE_HOSTPORT --os-image-api-version 1 image set $(basename "$MAGNUM_GUEST_IMAGE_URL" ".qcow2") $magnum_image_property
}
</code></pre>
安装magnum客户端
<pre><code>
function install_magnumclient {
    if use_library_from_git "python-magnumclient"; then
        git_clone_by_name "python-magnumclient"
        setup_dev_lib "python-magnumclient"
    fi
}
</code></pre>
启动magnum服务，传递port，protocol，tls等信息。进程直接通信需要tls安全传输层协议  
<pre><code>
function start_magnum_api {
    # Get right service port for testing
    local service_port=$MAGNUM_SERVICE_PORT
    local service_protocol=$MAGNUM_SERVICE_PROTOCOL
    if is_service_enabled tls-proxy; then
        service_port=$MAGNUM_SERVICE_PORT_INT
        service_protocol="http"
    fi
</code></pre>
为了满足进程之间通信。还需要对iptables进行配置。对keystone和magnum的通信进行accept  
<pre><code>
function configure_iptables {
    if [ "$MAGNUM_CONFIGURE_IPTABLES" != "False" ]; then
        ROUTE_TO_INTERNET=$(ip route get 8.8.8.8)
        OBOUND_DEV=$(echo ${ROUTE_TO_INTERNET#*dev} | awk '{print $1}')
        sudo iptables -t nat -A POSTROUTING -o $OBOUND_DEV -j MASQUERADE
        # bay nodes will access magnum-api (port $MAGNUM_SERVICE_PORT) to get CA certificate.
        sudo iptables -I INPUT -d $HOST_IP -p tcp --dport $MAGNUM_SERVICE_PORT -j ACCEPT || true
        sudo iptables -I INPUT -d $HOST_IP -p tcp --dport $KEYSTONE_SERVICE_PORT -j ACCEPT || true
    fi
}
</code></pre>
在plugin.sh中如果magnum的api和conduct服务启动，那么将会安装magnum和magnum-client，以及获取magnum_image等操作。  
另外对keystone的配置文件进行修改，创建magnum的account
<pre><code>
elif [[ "$1" == "stack" && "$2" == "post-config" ]]; then
        echo_summary "Configuring magnum"
        configure_magnum

        # Hack a large timeout for now
        iniset /etc/keystone/keystone.conf token expiration 7200

        if is_service_enabled key; then
            create_magnum_accounts
        fi
</code></pre>
在settings中则为一系列配置参数。用于服务的开启和关闭
<pre><code>
# Enable Neutron which is required by Magnum and disable nova-network.
disable_service n-net
enable_service q-svc
enable_service q-agt
enable_service q-dhcp
enable_service q-l3
enable_service q-meta
# Note: Default template uses LBaaS.
enable_service q-lbaas
enable_service neutron

# Enable Heat services
enable_service h-eng
enable_service h-api
enable_service h-api-cfn
enable_service h-api-cw
</code></pre>
devstack的代码只是一小部分。不过也能从这里看出magnum是如何运行的，在OpenStack的峰会上容器越来越火，看好Kolla，magnum以及Murano。  