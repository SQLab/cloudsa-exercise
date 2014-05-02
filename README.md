Cloud System Administration Exercise
====================================

Exercise One
------------

### Notice

This is a quick guide for ~~brainlessly~~ building a minimal OpenStack multi-node environment.
Always refer to http://docs.openstack.org/ for more details.

### Overview

An example set up of two-node architecture

* Controller Node
    - Identity (Keystone)
    - Image Service (Glance)
    - Compute Management (Nova)
    - Networking Management (Nova Network)
    - Dashboard (Horizon)
* Compute Node
    - Compute (Nova)
    - Networking (Nova Network)

### Prerequisite

Deploy three Ubuntu 14.04 LTS (Trust Tahr) nodes on
https://openstack-grizzly.it.nctu.edu.tw, one controller node and two compute nodes.

Sample profile

* flavor: cpu1ram1disk20
* image: Ubuntu 14.04 LTS (Trusty Tahr) amd64 cloudimg
* networking: Shared_Network

Heads up! Ubuntu cloud image disables password login by default, fill the
following cloud-init config into `Customization Script` on `Post-Creation` tab
to enable login through password.

```
#cloud-config
password: <your favorite password ex. 123456>
chpasswd: { expire: False }
ssh_pwauth: True
```

### General setup

#### Change to faster mirror site

```bash
$ sudo sed -i 's/nova.clouds.archive.ubuntu.com/ubuntu.cs.nctu.edu.tw/g' /etc/apt/sources.list
$ sudo sed -i 's/security.ubuntu.com/ubuntu.cs.nctu.edu.tw/g' /etc/apt/sources.list
```

#### Update package database

```
$ sudo apt-get update
```

#### NTP

##### Install ntp
```bash
$ sudo apt-get install ntp
```
##### Start ntp daemon
```bash
$ sudo service ntp start
```

#### Setup `/etc/hosts`
```
<CONTROLLER_IP> controller
<COMPUTE1_IP> compute1
<COMPUTE2_IP> compute2
```

### Controller Node

#### Database

##### Install package(s)
```bash
$ sudo apt-get install python-mysqldb mysql-server
```

##### Modify `/etc/mysql/my.cnf`
```
[mysqld]
...
bind-address        = <CONTROLLER_IP>
...
default-storage-engine = innodb
collation-server = utf8_general_ci
init-connect = 'SET NAMES utf8'
character-set-server = utf8
```

##### Restart mysql daemon
```bash
$ sudo service mysql restart
```

##### Setup MySQL
You could answer `Y` to most of the questions.
```bash
$ mysql_secure_installation
```

#### Messaging Server

##### Install RabbitMQ server
```bash
$ sudo apt-get install rabbitmq-server
```

##### Change password for user `guest`
```bash
$ sudo rabbitmqctl change_password guest <RABBIT_PASS>
```

#### Identity Service (Keystone)

##### Install package(s)
```bash
$ sudo apt-get install keystone
```

##### Modify `/etc/keystone/keystone.conf`
```
[DEFAULT]
# A "shared secret" between keystone and other openstack services
admin_token = <ADMIN_TOKEN>
...
log_dir = /var/log/keystone
...
[database]
# The SQLAlchemy connection string used to connect to the database
connection = mysql://keystone:<KEYSTONE_DBPASS>@controller/keystone
...
```

##### Setup keystone database
```bash
$ mysql -u root -p
mysql> CREATE DATABASE keystone;
mysql> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY '<KEYSTONE_DBPASS>';
mysql> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY '<KEYSTONE_DBPASS>';
mysql> exit
```

##### Initialize keystone database
```bash
$ sudo -u keystone keystone-manage db_sync
```

##### Restart keystone service
```bash
$ sudo service keystone restart
```

##### Define users, tenants and roles
```bash
$ export OS_SERVICE_TOKEN=<ADMIN_TOKEN>
$ export OS_SERVICE_ENDPOINT=http://controller:35357/v2.0
# Admin user
$ keystone user-create --name=admin --pass=<ADMIN_PASS> --email=<ADMIN_EMAIL>
$ keystone role-create --name=admin
$ keystone tenant-create --name=admin --description="Admin Tenant"
$ keystone user-role-add --user=admin --tenant=admin --role=admin
$ keystone user-role-add --user=admin --tenant=admin --role=_member_
# Demo user
$ keystone user-create --name=demo --pass=<DEMO_PASS> --email=<DEMO_EMAIL>
$ keystone tenant-create --name=demo --description="Demo Tenant"
$ keystone user-role-add --user=demo --role=_member_ --tenant=demo
# Service tenant
$ keystone tenant-create --name=service --description="Service Tenant"
```

##### Define services and API endpoints
```bash
$ keystone service-create --name=keystone --type=identity --description="OpenStack Identity"
$ keystone endpoint-create \
  --service-id=$(keystone service-list | awk '/ identity / {print $2}') \
  --publicurl=http://controller:5000/v2.0 \
  --internalurl=http://controller:5000/v2.0 \
  --adminurl=http://controller:35357/v2.0
```

##### Create a basic rc file
This rc file will be useful later on. To use it, run `source adminrc`
```
export OS_USERNAME=admin
export OS_PASSWORD=<ADMIN_PASS>
export OS_TENANT_NAME=admin
export OS_AUTH_URL=http://controller:35357/v2.0
```

##### Verify keystone installation
Refer to http://docs.openstack.org/icehouse/install-guide/install/apt/content/keystone-verify.html

#### Image Service (Glance)

##### Install package(s)
```bash
$ sudo apt-get install glance python-glanceclient
```

##### Modify `/etc/glance/glance-api.conf`
```
[DEFAULT]
...
rpc_backend = rabbit
rabbit_host = controller
rabbit_password = <RABBIT_PASS>
...
[database]
connection = mysql://glance:<GLANCE_DBPASS>@controller/glance
...
[keystone_authtoken]
auth_uri = http://controller:5000
auth_host = controller
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = glance
admin_password = <GLANCE_PASS>
```

##### Modify `/etc/glance/glance-registry.conf`
```
...
[database]
connection = mysql://glance:<GLANCE_DBPASS>@controller/glance
...
[keystone_authtoken]
auth_uri = http://controller:5000
auth_host = controller
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = glance
admin_password = <GLANCE_PASS>
```

##### Setup glance database
```bash
$ mysql -u root -p
mysql> CREATE DATABASE glance;
mysql> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY '<GLANCE_DBPASS>';
mysql> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY '<GLANCE_DBPASS>';
mysql> exit
```

##### Initialize glance database
```bash
$ sudo -u glance glance-manage db_sync
```

##### Restart glance service
```bash
$ sudo service glance-api restart
$ sudo service glance-registry restart
```

##### Define glance user
```bash
$ export OS_SERVICE_TOKEN=<ADMIN_TOKEN>
$ export OS_SERVICE_ENDPOINT=http://controller:35357/v2.0
$ keystone user-create --name=glance --pass=<GLANCE_PASS> --email=<ADMIN_EMAIL>
$ keystone user-role-add --user=glance --tenant=service --role=admin
```

##### Define glance service and API endpoint
```bash
$ keystone service-create --name=glance --type=image --description="OpenStack Image Service"
$ keystone endpoint-create \
  --service-id=$(keystone service-list | awk '/ image / {print $2}') \
  --publicurl=http://controller:9292 \
  --internalurl=http://controller:9292 \
  --adminurl=http://controller:9292
```

##### Upload a testing image
```
$ source adminrc
$ glance image-create --name="cirros-0.3.2-x86_64" --disk-format=qcow2 \
  --container-format=bare --is-public=true \
  --copy-from http://cdn.download.cirros-cloud.net/0.3.2/cirros-0.3.2-x86_64-disk.img
$ glance image-list
```

##### Verify glance installation
Refer to http://docs.openstack.org/icehouse/install-guide/install/apt/content/glance-verify.html

#### Compute Service (Nova)

##### Install package(s)
```bash
$ sudo apt-get install nova-api nova-cert nova-conductor nova-consoleauth \
  nova-novncproxy nova-scheduler python-novaclient
```

##### Modify `/etc/nova/nova.conf`
```
[DEFAULT]
...
rpc_backend = rabbit
rabbit_host = controller
rabbit_password = <RABBIT_PASS>

my_ip = <CONTROLLER_IP>
vncserver_listen = <CONTROLLER_IP>
vncserver_proxyclient_address = <CONTROLLER_IP>

auth_strategy = keystone

[database]
connection = mysql://nova:<NOVA_DBPASS>@controller/nova

[keystone_authtoken]
auth_uri = http://controller:5000
auth_host = controller
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = nova
admin_password = <NOVA_PASS>
```

##### Setup MySQL database
```bash
$ mysql -u root -p
mysql> CREATE DATABASE nova;
mysql> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY '<NOVA_DBPASS>';
mysql> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY '<NOVA_DBPASS>';
mysql> exit
```

##### Initialize nova database
```bash
$ sudo -u nova nova-manage db sync
```

##### Restart nova service
```bash
$ sudo service nova-api restart
$ sudo service nova-cert restart
$ sudo service nova-consoleauth restart
$ sudo service nova-scheduler restart
$ sudo service nova-conductor restart
$ sudo service nova-novncproxy restart
```

##### Define nova user
```bash
$ export OS_SERVICE_TOKEN=<ADMIN_TOKEN>
$ export OS_SERVICE_ENDPOINT=http://controller:35357/v2.0
$ keystone user-create --name=nova --pass=<NOVA_PASS> --email=<ADMIN_EMAIL>
$ keystone user-role-add --user=nova --tenant=service --role=admin
```

##### Define nova service and API endpoint
```bash
$ keystone service-create --name=nova --type=compute --description="OpenStack Compute"
$ keystone endpoint-create \
  --service-id=$(keystone service-list | awk '/ compute / {print $2}') \
  --publicurl=http://controller:8774/v2/%\(tenant_id\)s \
  --internalurl=http://controller:8774/v2/%\(tenant_id\)s \
  --adminurl=http://controller:8774/v2/%\(tenant_id\)s
```

#### Networking Service (Nova Network)

##### Modify `/etc/nova/nova.conf`
```
[DEFAULT]
...
network_api_class = nova.network.api.API
security_group_api = nova
```

##### Restart nova service
```bash
$ sudo service nova-api restart
$ sudo service nova-scheduler restart
$ sudo service nova-conductor restart
```

##### Create a flat network
```
$ source adminrc
$ nova network-create demo-net --bridge br100 --multi-host T \
  --fixed-range-v4 192.168.100.0/24
$ nova net-list
```

#### Dashboard (Horizon)

##### Install package(s)
```bash
$ sudo apt-get install apache2 memcached libapache2-mod-wsgi openstack-dashboard
```

### Compute Node(s)

#### Database
```bash
$ sudo apt-get install python-mysqldb
```

#### Compute Service (Nova)

##### Install package(s)
```bash
$ sudo apt-get install nova-compute-kvm python-guestfs
```

##### Make kernel readable by normal user
```bash
$ sudo dpkg-statoverride  --update --add root root 0644 /boot/vmlinuz-$(uname -r)
```

##### Modify `/etc/nova/nova.conf`
```
[DEFAULT]
...
auth_strategy = keystone
...
rpc_backend = rabbit
rabbit_host = controller
rabbit_password = <RABBIT_PASS>
...
my_ip = <COMPUTEN_IP>
vnc_enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = <COMPUTEN_IP>
novncproxy_base_url = http://controller:6080/vnc_auto.html
...
glance_host = controller
...
[database]
# The SQLAlchemy connection string used to connect to the database
connection = mysql://nova:<NOVA_DBPASS>@controller/nova

[keystone_authtoken]
auth_uri = http://controller:5000
auth_host = controller
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = nova
admin_password = <NOVA_PASS>
```

##### Modify `/etc/nova/nova-compute.conf`
This testing environment does not support nested hardware acceleration
```
[libvirt]
...
virt_type = qemu
```

##### Restart nova service
```bash
$ sudo service nova-compute restart
```

#### Networking Service (Nova Network)

##### Install package(s)
```bash
$ sudo apt-get install nova-network nova-api-metadata
```

##### Modify `/etc/nova/nova.conf`
```
[DEFAULT]
...
network_api_class = nova.network.api.API
security_group_api = nova
firewall_driver = nova.virt.libvirt.firewall.IptablesFirewallDriver
network_manager = nova.network.manager.FlatDHCPManager
network_size = 254
allow_same_net_traffic = False
multi_host = True
send_arp_for_ha = True
share_dhcp_address = True
force_dhcp_release = True
flat_network_bridge = br100
flat_interface = eth0
public_interface = br100
```

##### Restart nova-network
```
$ sudo service nova-network restart
$ sudo service nova-api-metadata restart
```
