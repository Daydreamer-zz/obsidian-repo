# Openstack安装

当前openstack安装的版本是train，操作系统是centos7.9

主机规划

| IP            | 主机名     | 角色         |
| ------------- | ---------- | ------------ |
| 192.168.10.8  | controller | 控制节点     |
| 192.168.10.9  | compute1   | 计算节点1    |
| 192.168.10.10 | compute2   | 计算节点2    |
| 192.168.10.11 | cinder     | cinder块存储 |

## 1.系统优化

### 1.1 禁用服务和优化

```bash
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config
systemctl disable firewalld --now
systemctl disable NetworkManager --now
chmod +x /etc/rc.d/rc.local
systemctl list-unit-files|egrep "^ab|^aud|^kdump|vm|^md|^mic|^post|lvm"  |awk '{print $1}'|sed -r 's#(.*)#systemctl disable &#g'|bash
echo 'export HISTTIMEFORMAT="%Y-%m-%d %H:%M:%S  "' >> /etc/profile
source /etc/profile
sed -i "s/^#UseDNS yes/UseDNS no/g" /etc/ssh/sshd_config
sed -i "s/GSSAPIAuthentication yes/GSSAPIAuthentication no/g" /etc/ssh/sshd_config
systemctl restart sshd
```

### 1.2 时间同步配置文件

```bash
cat << EOF > /etc/chrony.conf
server ntp.aliyun.com iburst
stratumweight 0
driftfile /var/lib/chrony/drift
rtcsync
makestep 10 3
bindcmdaddress 127.0.0.1
bindcmdaddress ::1
keyfile /etc/chrony.keys
commandkey 1
generatecommandkey
logchange 0.5
logdir /var/log/chrony
EOF
```

### 1.3 limits

```bash
cat << EOF >> /etc/security/limits.conf
* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited
EOF
```

### 1.4 .yum源配置

```bash
sed -i "s#enabled=1#enabled=0#g" /etc/yum/pluginconf.d/fastestmirror.conf
sed -i "s#plugins=1#plugins=0#g" /etc/yum.conf
```

Base

```bash
sed -e 's|^mirrorlist=|#mirrorlist=|g' \
-e 's|^#baseurl=http://mirror.centos.org|baseurl=http://mirror.nju.edu.cn|g' \
-i /etc/yum.repos.d/CentOS-*.repo
```

Openstack

```bash
cat << \EOF > /etc/yum.repos.d/CentOS-Ceph-Nautilus.repo
[centos-ceph-nautilus]
name=CentOS-$releasever - Ceph Nautilus
baseurl=http://mirrors.nju.edu.cn/$contentdir/$releasever/storage/$basearch/ceph-nautilus/
gpgcheck=0
enabled=1
EOF

cat << \EOF > /etc/yum.repos.d/CentOS-NFS-Ganesha-28.repo
[centos-nfs-ganesha28]
name=CentOS-$releasever - NFS Ganesha 2.8
baseurl=https://mirrors.nju.edu.cn/$contentdir/$releasever/storage/$basearch/nfs-ganesha-28/
gpgcheck=0
enabled=1
EOF


cat << \EOF > /etc/yum.repos.d/CentOS-OpenStack-train.repo
[centos-openstack-train]
name=CentOS-7 - OpenStack train
baseurl=http://mirrors.nju.edu.cn/$contentdir/$releasever/cloud/$basearch/openstack-train/
gpgcheck=0
enabled=1
exclude=sip,PyQt4
EOF

cat << \EOF > /etc/yum.repos.d/CentOS-QEMU-EV.repo
[centos-qemu-ev]
name=CentOS-$releasever - QEMU EV
baseurl=http://mirrors.nju.edu.cn/$contentdir/$releasever/virt/$basearch/kvm-common/
gpgcheck=0
enabled=1
EOF
```

### 1.5 hosts

```bash
192.168.10.8 controller
192.168.10.9 compute1
```

### 1.6 安装openstack包

```bash
yum install -y wget vim net-tools chrony bash-completion lrzsz zip unzip tar bridge-utils libibverbs
```

```bash
yum install -y python-openstackclient openstack-utils
```

## 2. 安装controller节点

### 2.1 安装控制节点相关包

```bash
yum install -y mariadb mariadb-server python2-PyMySQL rabbitmq-server memcached python-memcached openstack-keystone httpd mod_wsgi openstack-glance openstack-placement-api openstack-nova-api openstack-nova-conductor openstack-nova-novncproxy openstack-nova-scheduler openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables openstack-dashboard
```

### 2.2 配置mysql

配置mysql配置文件

```ini
[mysqld]
bind-address = 192.168.10.8
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

启动mysql

```bash
systemctl enable mariadb --now
```

初始化mysql

```bash
mysql_secure_installation
```

建库并授权

```sql
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';


CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';

CREATE DATABASE placement;
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'PLACEMENT_DBPASS';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'PLACEMENT_DBPASS';

CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';

CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'NEUTRON_DBPASS';
```

### 2.3 配置rabbitmq

启动rabbitmq

```bash
systemctl enable rabbitmq-server.service --now
```

创建用户并授权

```bash
rabbitmqctl add_user openstack RABBIT_PASS
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

### 2.4配置memcache

配置memcache

```bash
vim /etc/sysconfig/memcached

OPTIONS="-l 0.0.0.0"
```

启动memcache

```bash
systemctl enable memcached.service --now
```

### 2.5 配置keystone

修改**/etc/keystone/keystone.conf**

```bash
cat << \EOF > /etc/keystone/keystone.conf
[database]
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone

[token]
provider = fernet
EOF
```

同步数据库

```bash
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

初始化fernet

```bash
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

初始化keystone

```bash
keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```

配置httpd

```bash
sed -i "s/#ServerName www.example.com:80/ServerName controller/g" /etc/httpd/conf/httpd.conf
```

```bash
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
```

启动httpd

```bash
systemctl enable httpd --now
```

保存为环境变量文件

```bash
cat << \EOF > admin-openrc
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF
```

创建service

```bash
source admin-openrc
openstack project create --domain default --description "Service Project" service
```

### 2.6 配置glance

在keystone创建服务用户,并关联角色

```bash
openstack user create --domain default --password GLANCE_PASS  glance
openstack role add --project service --user glance admin
```

在keystone上注册api访问地址

```bash
openstack service create --name glance --description "OpenStack Image" image
openstack endpoint create --region RegionOne image public http://controller:9292
openstack endpoint create --region RegionOne image internal http://controller:9292
openstack endpoint create --region RegionOne image admin http://controller:9292
```

修改**/etc/glance/glance-api.conf**

```bash
cat << \EOF > /etc/glance/glance-api.conf
[database]
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images

[keystone_authtoken]
www_authenticate_uri  = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
flavor = keystone
EOF
```

同步数据库

```bash
su -s /bin/sh -c "glance-manage db_sync" glance
```

启动glance服务

```bash
systemctl enable openstack-glance-api.service --now
```

下载镜像测试上传

```bash
wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img

glance image-create --name "cirros" --file cirros-0.4.0-x86_64-disk.img --disk-format qcow2 --container-format bare --visibility public
```

### 2.7 配置placement

在keystone创建placement服务用户,并关联角色

```bash
openstack user create --domain default --password PLACEMENT_PASS placement
openstack role add --project service --user placement admin
```

在服务目录中创建 Placement API 条目和API服务端点：

```bash
openstack service create --name placement --description "Placement API" placement
openstack endpoint create --region RegionOne placement public http://controller:8778
openstack endpoint create --region RegionOne placement internal http://controller:8778
openstack endpoint create --region RegionOne placement admin http://controller:8778
```

编辑**/etc/placement/placement.conf**

```bash
cat <<\EOF > /etc/placement/placement.conf
[placement_database]
connection = mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement

[api]
auth_strategy = keystone

[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = PLACEMENT_PASS
EOF
```

编辑**/etc/httpd/conf.d/00-placement-api.conf**

VirtualHost上面加入如下内容<span style="color:red">**必须操作这里，否则实例无法调度**</span>.

```xml
<Directory /usr/bin>
  Require all denied
  <Files "placement-api">
    <RequireAll>
      Require all granted
      Require not env blockAccess
    </RequireAll>
  </Files>
</Directory>
```

同步数据库

```bash
su -s /bin/sh -c "placement-manage db sync" placement
```

重启httpd

```bash
systemctl restart httpd
```

 ### 2.8 配置nova

在keystone创建服务用户,并关联角色

```bash
openstack user create --domain default --password NOVA_PASS  nova
openstack role add --project service --user nova admin
```

在keystone上注册api访问地址

```bash
openstack service create --name nova --description "OpenStack Compute" compute
openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1
```

修改**/etc/nova/nova.conf**

```bash
cat <<\EOF > /etc/nova/nova.conf
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:RABBIT_PASS@controller:5672/
my_ip = 192.168.10.8
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api]
auth_strategy = keystone

[api_database]
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api

[database]
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova

[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = NOVA_PASS

[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip

[glance]
api_servers = http://controller:9292

[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = PLACEMENT_PASS

[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
service_metadata_proxy = true
metadata_proxy_shared_secret = METADATA_SECRET

[oslo_concurrency]
lock_path = /var/lib/nova/tmp
EOF
```

同步数据库

```bash
su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
su -s /bin/sh -c "nova-manage db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
```

启动服务

```bash
systemctl enable openstack-nova-api.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service --now
```

### 2.9 配置neutron

这里网络配置先安装公共网络(Provider networks)类型

在keystone创建服务用户,并关联角色

```bash
openstack user create --domain default --password NEUTRON_PASS  neutron
openstack role add --project service --user neutron admin
```

在keystone上注册api访问地址

```bash
openstack service create --name neutron --description "OpenStack Networking" network 
openstack endpoint create --region RegionOne network public http://controller:9696
openstack endpoint create --region RegionOne network internal http://controller:9696
openstack endpoint create --region RegionOne network admin http://controller:9696
```

修改**/etc/neutron/neutron.conf**

```bash
cat << \EOF > /etc/neutron/neutron.conf
[DEFAULT]
core_plugin = ml2
service_plugins =
transport_url = rabbit://openstack:RABBIT_PASS@controller
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[database]
connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_PASS

[nova]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = NOVA_PASS

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
EOF
```

配置**/etc/neutron/plugins/ml2/ml2_conf.ini**

```bash
cat << \EOF > /etc/neutron/plugins/ml2/ml2_conf.ini
[ml2]
type_drivers = flat,vlan
tenant_network_types =
mechanism_drivers = linuxbridge
extension_drivers = port_security

[ml2_type_flat]
flat_networks = provider

[securitygroup]
enable_ipset = true
EOF
```

配置**/etc/neutron/plugins/ml2/linuxbridge_agent.ini**

```bash
cat << \EOF > /etc/neutron/plugins/ml2/linuxbridge_agent.ini
[linux_bridge]
physical_interface_mappings = provider:eth0

[vxlan]
enable_vxlan = false

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
EOF
```

配置**/etc/neutron/dhcp_agent.ini**

```bash
cat << \EOF > /etc/neutron/dhcp_agent.ini
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
EOF
```

配置**/etc/neutron/metadata_agent.ini**

```bash
cat << \EOF > /etc/neutron/metadata_agent.ini
[DEFAULT]
nova_metadata_host = controller
metadata_proxy_shared_secret = METADATA_SECRET
EOF
```

创建软链接

```bash
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

同步数据库

```bash
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

重启nova-api服务

```bash
systemctl restart openstack-nova-api.service
```

启动neutron服务

```bash
systemctl enable neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service --now
```

### 2.10 配置horizon

配置参考官方文档https://docs.openstack.org/horizon/train/install/install-rdo.html

启动

```bash
systemctl restart httpd
```

## 3.安装计算节点

### 3.1 安装计算节点相关包

```bash
yum install -y openstack-nova-compute openstack-neutron-linuxbridge ebtables ipset
```

### 3.2 配置nova

修改**/etc/nova/nova.conf**

```bash
cat << \EOF > /etc/nova/nova.conf
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:RABBIT_PASS@controller
my_ip = 192.168.10.9
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = NOVA_PASS

[glance]
api_servers = http://controller:9292

[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = PLACEMENT_PASS

[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[libvirt]
virt_type = kvm

[scheduler]
discover_hosts_in_cells_interval = 300

[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html
EOF
```

### 3.3 配置neutron

修改**/etc/neutron/neutron.conf**

```bash
cat << \EOF > /etc/neutron/neutron.conf
[DEFAULT]
transport_url = rabbit://openstack:RABBIT_PASS@controller
auth_strategy = keystone


[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_PASS

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
EOF
```

修改**/etc/neutron/plugins/ml2/linuxbridge_agent.ini**

```bash
cat << \EOF > /etc/neutron/plugins/ml2/linuxbridge_agent.ini
[linux_bridge]
physical_interface_mappings = provider:eth0

[vxlan]
enable_vxlan = false

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
EOF
```

### 3.4 启动计算节点所有服务

```bash
systemctl enable neutron-linuxbridge-agent.service openstack-nova-compute.service libvirtd --now
```

## 4.控制节点发现计算节点

### 4.1 发现计算节点

```bash
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
```

### 4.2 查看计算节点

```bash
openstack compute service list
```

```bash
openstack hypervisor list
```

## 5.创建实例测试

### 5.1 创建网络

```bash
openstack network create  --share --external \
  --provider-physical-network provider \
  --provider-network-type flat wan
```

### 5.2 创建子网

```bash
openstack subnet create --network wan \
  --allocation-pool start=192.168.10.20,end=192.168.10.80 \
  --dns-nameserver 192.168.10.1 --gateway 192.168.10.1 \
  --subnet-range 192.168.10.0/24 192.168.10.0
```

### 5.3 创建虚拟机硬件配置方案

```bash
openstack flavor create --id 0 --vcpus 1 --ram 64 --disk 1 m1.nano
```

### 5.4 创建sshkey

```bash
openstack keypair create --public ~/.ssh/id_rsa.pub mykey
```

### 5.5 启动实例

**这里的netid必须和之前创建的一致，不要复制这里的**

```bash
openstack server create --flavor m1.nano --image cirros \
  --nic net-id=0d517510-2e7e-4be4-a948-d0e4b207b2cd --security-group default \
  --key-name mykey oldboy
```

### 5.6 实例启动错误解决

对于Vmware Workstation下的宿主机，实例可能会无法启动，需要编辑**/etc/nova/nova.conf**加入

```ini
[DEFAULT]
compute_driver = libvirt.LibvirtDriver

[libvirt]
virt_type=kvm
cpu_mode=host-passthrough
hw_machine_type = x86_64=pc-i440fx-rhel7.2.0
```

重启计算节点nova-compute服务

```bash
systemctl restart openstack-nova-compute.service
```

## 6.安装cinder块存储服务

### 6.1 安装配置控制节点

安装软件包

```bash
yum install -y openstack-cinder
```

创建数据库

```sql
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'CINDER_DBPASS';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'CINDER_DBPASS';
```

创建cinder用户并关联角色

```shell
openstack user create --domain default --password CINDER_PASS cinder
openstack role add --project service --user cinder admin
```

创建服务实和服务端点

```bash
openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
openstack endpoint create --region RegionOne volumev2 public http://controller:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev2 internal http://controller:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev2 admin http://controller:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 public http://controller:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 internal http://controller:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 admin http://controller:8776/v3/%\(project_id\)s
```

配置**/etc/cinder/cinder.conf**

```bash
cat << \EOF > /etc/cinder/cinder.conf
[DEFAULT]
transport_url = rabbit://openstack:RABBIT_PASS@controller
auth_strategy = keystone
my_ip = 192.168.10.8

[database]
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = CINDER_PASS

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
EOF
```

同步数据库

```bash
su -s /bin/sh -c "cinder-manage db sync" cinder
```

配置**/etc/nova/nova.conf**加入

```bash
cat << \EOF >> /etc/nova/nova.conf

[cinder]
os_region_name = RegionOne
EOF
```

重启nova-api服务

```bash
systemctl restart openstack-nova-api.service
```

启动块存储服务并将它们配置为在系统启动时启动

```bash
systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service --now
```

### 6.2 安装和配置存储节点

这里存储节点添加一块盘，设备名为`/dev/sdb`，存储节点使用lvm提供服务

安装软件包

```bash
yum install -y lvm2 device-mapper-persistent-data openstack-cinder targetcli python-keystone
```

启动 LVM 元数据服务并将其配置为在系统引导时启动

```bash
systemctl enable lvm2-lvmetad.service --now
```

配置lvm卷

```bash
pvcreate /dev/sdb
vgcreate cinder-volumes /dev/sdb
```

配置**/etc/lvm/lvm.conf**

在该`devices`部分中，添加一个接受 `/dev/sdb`设备并拒绝所有其他设备的过滤器

```
devices {
...
filter = [ "a/sdb/", "r/.*/"]
```

配置**/etc/cinder/cinder.conf**

```bash
cat << \EOF > /etc/cinder/cinder.conf
[DEFAULT]
transport_url = rabbit://openstack:RABBIT_PASS@controller
auth_strategy = keystone
my_ip = 192.168.10.10
enabled_backends = lvm
glance_api_servers = http://controller:9292

[database]
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = CINDER_PASS

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp

[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
target_protocol = iscsi
target_helper = lioadm
EOF
```

启动块存储卷服务，包括其依赖项，并将它们配置为在系统启动时启动

```bash
systemctl enable openstack-cinder-volume.service target.service --now
```

## 7.配置neutron为私有网络类型

额外需要单独添加一块网卡eth1

### 7.1 配置控制节点

在之前安装的公共网络基础上修改**/etc/neutron/neutron.conf**

```ini
[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
```

在之前的基础上修改**/etc/neutron/plugins/ml2/ml2_conf.ini**

```ini
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population

[ml2_type_vxlan]
vni_ranges = 1:1000
```

在之前基础上修改**/etc/neutron/plugins/ml2/linuxbridge_agent.ini**

```ini
[vxlan]
enable_vxlan = true
local_ip = 192.168.2.11
l2_population = true
```

编辑**/etc/neutron/l3_agent.ini**

```ini
[DEFAULT]
interface_driver = linuxbridge
```

重启服务

```bash
systemctl restart  neutron-server.service   neutron-linuxbridge-agent.service neutron-dhcp-agent.service   neutron-metadata-agent.service
```

启动l3服务

```bash
systemctl enable neutron-l3-agent.service --now
```

### 7.2 配置计算节点

在之前的安装的基础上配置**/etc/neutron/plugins/ml2/linuxbridge_agent.ini**

```ini
[vxlan]
enable_vxlan = true
local_ip = 192.168.2.12
l2_population = true
```

重启服务

```bash
systemctl restart neutron-linuxbridge-agent.service
```

### 7.3 启动创建网络

