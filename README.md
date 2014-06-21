Cloud System Administration Exercise
====================================

Notation
--------

### User-defined variable

User-defined variable is documented in the form `<blah blah blah>`.
Replace it with your choice. For example, replace `<CONTROLLER_IP>` with the
IP address of your controller node. And replace `<KEYSTONE_DBPASS>` with your
favorite password.

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

* flavor:
    - controller node: cpu1ram2disk20
    - others: cpu1ram1disk20
* image: Ubuntu 14.04 LTS (Trusty Tahr) amd64 cloudimg
* networking: Shared_Network

Heads up! Ubuntu cloud image disables password login by default, fill the
following cloud-init config into `Customization Script` on `Post-Creation` tab
to enable login through password.

```
#cloud-config
password: <USER_PASSWORD>
chpasswd: { expire: False }
ssh_pwauth: True
```

Once the instance boots up, you should be able to login with user `ubuntu`
and password `<USER_PASSWORD>`.

### General setup

This section applys to all nodes.

#### Switch to faster mirror site

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

#### Associate a pulic ip address

Associate a public ip address for the controller node via the website.
You will be able to access the controller node through this ip address,
which means your servers are exposed to the wild without any protection.
You better use stronger password and set up a firewall service.

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
  --publicurl=http://<CONTROLLER_PUBLIC_IP>:5000/v2.0 \
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
  --publicurl=http://<CONTROLLER_PUBLIC_IP>:9292 \
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
  --publicurl=http://<CONTROLLER_PUBLIC_IP>:8774/v2/%\(tenant_id\)s \
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
novncproxy_base_url = http://<CONTROLLER_PUBLIC_IP>:6080/vnc_auto.html
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

##### Create a flat network
```
controller $ source adminrc
controller $ nova network-create demo-net --bridge br100 --multi-host T \
  --fixed-range-v4 192.168.100.0/24
controller $ nova net-list
```

Exercise Two
------------

### Notice

Delete all nodes used in the previous exercise.

### Overview

An example set up of two-node architecture deployed by an extra puppet master
node.

* Puppet Master
    - Puppet Master
* Controller Node
    - Puppet Agent
    - Identity (Keystone)
    - Image Service (Glance)
    - Compute Management (Nova)
    - Networking Management (Nova Network)
    - Dashboard (Horizon)
* Compute Node
    - Puppet Agent
    - Compute (Nova)
    - Networking (Nova Network)

### Prerequisite

Sample profile

* flavor:
    - puppet master: m1.tiny
    - controller node: cpu1ram2disk20
    - others: cpu1ram1disk20
* image: Ubuntu 14.04 LTS (Trusty Tahr) amd64 cloudimg
* networking: Shared_Network

### General Setup

This section applys to all nodes.

#### Switch to faster mirror site

```bash
$ sudo sed -i 's/nova.clouds.archive.ubuntu.com/ubuntu.cs.nctu.edu.tw/g' /etc/apt/sources.list
$ sudo sed -i 's/security.ubuntu.com/ubuntu.cs.nctu.edu.tw/g' /etc/apt/sources.list
```

#### Update package database
```bash
$ sudo apt-get update
```

#### NTP

##### Install ntp
```bash
$ sudo apt-get install ntp
```

#### Setup `/etc/hosts`
```
<PUPPETMASTER_IP> puppet
<CONTROLLER_IP> controller
<COMPUTE1_IP> compute1
<COMPUTE2_IP> compute2
```

### Puppet Master

#### Puppet

##### Install package(s)
```bash
$ sudo apt-get install puppetmaster
```

##### Install openstack puppet module
```bash
$ sudo puppet module install puppetlabs-openstack
```

##### Patch puppetlabs-mysql

XXX: This is a workaround for Ubuntu 14.04. puppetlabs-mysql upstream has
already fixed the problem, please refer to puppetlabs/puppetlabs-mysql@c57c762

```bash
$ sudo sed -i 's/libmysql-ruby/ruby-mysql/' /etc/puppet/modules/mysql/manifests/params.pp
$ sudo sed -i 's/libmysql-ruby/ruby-mysql/' /etc/puppet/modules/mysql/spec/system/mysql_bindings_spec.rb
$ sudo sed -i 's/libmysql-ruby/ruby-mysql/' /etc/puppet/modules/mysql/spec/classes/mysql_bindings_spec.rb
$ sudo sed -i 's/libmysql-ruby/ruby-mysql/' /etc/puppet/modules/mysql/spec/acceptance/mysql_bindings_spec.rb
```

##### Create `/etc/puppet/hiera.yaml`
```yaml
---
:backends:
  - yaml
:yaml:
  :datadir: /etc/puppet/hieradata
:hierarchy:
  - common
```

##### Create directory `/etc/puppet/hieradata/`
```bash
$ sudo mkdir /etc/puppet/hieradata/
```

##### Create `/etc/puppet/hieradata/common.yaml`
```yaml
openstack::region: 'openstack'

######## Networks
openstack::network::api: '10.10.0.0/16'
openstack::network::external: '192.168.100.0/24'
openstack::network::management: '10.10.0.0/16'
openstack::network::data: '10.10.0.0/16'

openstack::network::external::ippool::start: 192.168.100.100
openstack::network::external::ippool::end: 192.168.100.250
openstack::network::external::gateway: 192.168.100.254
openstack::network::external::dns: 8.8.8.8

######## Private Neutron Network

openstack::network::neutron::private: '192.168.200.0/24'

######## Fixed IPs (controllers)

openstack::controller::address::api: '<CONTROLLER_IP>'
openstack::controller::address::management: '<CONTROLLER_IP>'
openstack::controller::address::api::public: '<CONTROLLER_PUBLIC_IP>'
openstack::storage::address::api: '<CONTROLLER_IP>'
openstack::storage::address::management: '<CONTROLLER_IP>'

######## Database

openstack::mysql::root_password: '<MYSQL_ROOT_PASSWORD>'
openstack::mysql::service_password: '<MYSQL_SERVICE_PASSWORD>'
openstack::mysql::allowed_hosts: ['localhost', '127.0.0.1', '10.10.0.%']

######## RabbitMQ

openstack::rabbitmq::user: 'openstack'
openstack::rabbitmq::password: '<RABBIT_PASS>'

######## Keystone

openstack::keystone::admin_token: '<ADMIN_TOKEN>'
openstack::keystone::admin_email: 'admin@example.com'
openstack::keystone::admin_password: '<KEYSTONE_PASSWORD>'

openstack::tenants:
    "test":
        description: "Test tenant"
    "demo":
        description: "Demo Tenant"
    "demo2":
        description: "Demo2 Tenant"
    "service":
        description: "Service Tenant"

openstack::users:
    "test":
        password: "<TEST_PASS>"
        tenant: "test"
        email: "test@example.com"
        admin: true
    "demo":
        password: "<DEMO_PASS>"
        tenant: "demo"
        email: "demo@example.com"
        admin: false
    "demo2":
        password: "<DEMO2_PASS>"
        tenant: "demo2"
        email: "demo2@example.com"
        admin: false

######## Glance

openstack::glance::password: '<GLANCE_PASS>'

######## Cinder

openstack::cinder::password: '<CINDER_PASS>'
openstack::cinder::volume_size: '8G'

######## Swift

openstack::swift::password: '<SWIFT_PASS>'
openstack::swift::hash_suffix: 'pop-bang'

######## Nova

openstack::nova::libvirt_type: 'kvm'
openstack::nova::password: '<NOVA_PASS>'

######## Neutron

openstack::neutron::password: '<NEUTRON_PASS>'
openstack::neutron::shared_secret: '<NEUTRON_SECRET>'

######## Ceilometer
openstack::ceilometer::mongo::password: '<CEILOMETER_MONGO_PASS>'
openstack::ceilometer::password: '<CEILOMETER_PASS>'
openstack::ceilometer::meteringsecret: '<CEILOMETER_SECRET_KEY>'

######## Heat
openstack::heat::password: '<HEAT_PASS>'
openstack::heat::encryption_key: '<HEAT_SECRET_KEY>'

######## Horizon

openstack::horizon::secret_key: '<HORIZON_SECRET_KEY>'

######## Tempest

openstack::tempest::configure_images    : true
openstack::tempest::image_name          : 'Cirros'
openstack::tempest::image_name_alt      : 'Cirros'
openstack::tempest::username            : 'demo'
openstack::tempest::username_alt        : 'demo2'
openstack::tempest::username_admin      : 'test'
openstack::tempest::configure_network   : true
openstack::tempest::public_network_name : 'public'
openstack::tempest::cinder_available    : false
openstack::tempest::glance_available    : true
openstack::tempest::horizon_available   : true
openstack::tempest::nova_available      : true
openstack::tempest::neutron_available   : false
openstack::tempest::heat_available      : false
openstack::tempest::swift_available     : false

######## Log levels
openstack::verbose: 'True'
openstack::debug: 'True'
```

##### Restart puppetmaster
```bash
$ sudo service puppetmaster restart
```

##### Create `/etc/puppet/modules/myopenstack/manifests/common/nova.pp`
```puppet
class myopenstack::common::nova ($is_compute    = false) {
  $is_controller = $::openstack::profile::base::is_controller

  $management_network = hiera('openstack::network::management')
  $management_address = ip_for_network($management_network)

  $storage_management_address = hiera('openstack::storage::address::management')
  $controller_management_address = hiera('openstack::controller::address::management')

  class { '::nova':
    sql_connection     => $::openstack::resources::connectors::nova,
    glance_api_servers => "http://${storage_management_address}:9292",
    memcached_servers  => ["${controller_management_address}:11211"],
    rabbit_hosts       => [$controller_management_address],
    rabbit_userid      => hiera('openstack::rabbitmq::user'),
    rabbit_password    => hiera('openstack::rabbitmq::password'),
    debug              => hiera('openstack::debug'),
    verbose            => hiera('openstack::verbose'),
    mysql_module       => '2.2',
  }

  nova_config {
    'DEFAULT/default_floating_pool': value => 'public';
    'DEFAULT/network_api_class':     value => 'nova.network.api.API';
    'DEFAULT/security_group_api':    value => 'nova';
  }

  if $is_controller {
    class { '::nova::api':
      admin_password                       => hiera('openstack::nova::password'),
      auth_host                            => $controller_management_address,
      enabled                              => $is_controller,
    }
  }

  class { '::nova::vncproxy':
    host    => hiera('openstack::controller::address::api'),
    enabled => $is_controller,
  }

  class { [
    'nova::scheduler',
    'nova::objectstore',
    'nova::cert',
    'nova::consoleauth',
    'nova::conductor'
  ]:
    enabled => $is_controller,
  }

  # TODO: it's important to set up the vnc properly
  class { '::nova::compute':
    enabled                       => $is_compute,
    vnc_enabled                   => true,
    vncserver_proxyclient_address => $management_address,
    vncproxy_host                 => hiera('openstack::controller::address::api::public'),
  }
}
```

##### Create `/etc/puppet/modules/myopenstack/manifests/profile/nova/api.pp`
```puppet
class myopenstack::profile::nova::api {
  openstack::resources::controller { 'nova': }
  openstack::resources::database { 'nova': }
  openstack::resources::firewall { 'Nova API': port => '8774', }
  openstack::resources::firewall { 'Nova Metadata': port => '8775', }
  openstack::resources::firewall { 'Nova EC2': port => '8773', }
  openstack::resources::firewall { 'Nova S3': port => '3333', }
  openstack::resources::firewall { 'Nova novnc': port => '6080', }

  class { '::nova::keystone::auth':
    password         => hiera('openstack::nova::password'),
    public_address   => hiera('openstack::controller::address::api::public'),
    admin_address    => hiera('openstack::controller::address::management'),
    internal_address => hiera('openstack::controller::address::management'),
    region           => hiera('openstack::region'),
    cinder           => true,
  }

  include ::myopenstack::common::nova
}
```

##### Create `/etc/puppet/modules/myopenstack/manifests/profile/nova/compute.pp`
```puppet
class myopenstack::profile::nova::compute {
  $management_network = hiera('openstack::network::management')
  $management_address = ip_for_network($management_network)

  class { 'myopenstack::common::nova':
    is_compute => true,
  }

  class { '::nova::compute::libvirt':
    libvirt_type     => hiera('openstack::nova::libvirt_type'),
    vncserver_listen => $management_address,
  }

  file { '/etc/libvirt/qemu.conf':
    ensure => present,
    source => 'puppet:///modules/openstack/qemu.conf',
    mode   => '0644',
    notify => Service['libvirt'],
  }

  Package['libvirt'] -> File['/etc/libvirt/qemu.conf']
}
```

##### Create `/etc/puppet/modules/myopenstack/manifests/profile/nova/network.pp`
```puppet
class myopenstack::profile::nova::network {
  class { '::nova::network':
    private_interface => 'eth0',
    fixed_range       => '192.168.100.0/24',
    public_interface  => 'br100',
    enabled           => true,
  }

  nova::generic_service { 'api-metadata':
    enabled        => true,
    manage_service => true,
    ensure_package => true,
    package_name   => 'nova-api-metadata',
    service_name   => 'nova-api-metadata',
  }
}
```

##### Create `/etc/puppet/modules/myopenstack/manifests/profile/horizon.pp`
```puppet
class myopenstack::profile::horizon {
  class { '::horizon':
    fqdn            => [ '127.0.0.1', hiera('openstack::controller::address::api'), hiera('openstack::controller::address::api::public'), $::fqdn ],
    secret_key      => hiera('openstack::horizon::secret_key'),
    cache_server_ip => hiera('openstack::controller::address::management'),

  }

  openstack::resources::firewall { 'Apache (Horizon)': port => '80' }
  openstack::resources::firewall { 'Apache SSL (Horizon)': port => '443' }

  if $::selinux and str2bool($::selinux) != false {
    selboolean{'httpd_can_network_connect':
      value      => on,
      persistent => true,
    }
  }
}
```

##### Create `/etc/puppet/manifests/site.pp`
```puppet
class mycontroller inherits ::openstack::role {
  class { '::openstack::profile::firewall': }
  class { '::openstack::profile::rabbitmq': }
  class { '::openstack::profile::memcache': }
  class { '::openstack::profile::mysql': }
  class { '::openstack::profile::keystone': }
  class { '::openstack::profile::glance::api': } ->
  class { '::openstack::profile::glance::auth': }
  class { '::myopenstack::profile::nova::api': }
  class { '::myopenstack::profile::horizon': }
  class { '::openstack::profile::auth_file': }
  class { '::openstack::setup::cirros': }
}

class mycompute inherits ::openstack::role {
  class { '::openstack::profile::firewall': }
  class { '::myopenstack::profile::nova::compute': }
  class { '::myopenstack::profile::nova::network': }
}

node 'controller.openstacklocal' {
    include mycontroller
}

node /^compute\d+.openstacklocal$/ {
    include mycompute
}
```

### Controller Node

#### Puppet

##### Install package(s)
```bash
$ sudo apt-get install puppet
```

##### Enable puppet agent
```bash
$ sudo puppet agent --enable
```

##### Sign certificate
```bash
$ sudo puppet agent -t
# Apply following two commands on the puppet master node
puppet $ sudo puppet cert list
puppet $ sudo puppet cert sign controller.openstacklocal
```

##### Create `/root/.my.cnf`
```
[client]
user=root
host=localhost
password=''
socket=/var/run/mysqld/mysqld.sock
```

##### Initial run
```bash
$ sudo puppet agent -t
```

### Compute Node

#### Puppet

##### Install package(s)
```bash
$ sudo apt-get install puppet
```

##### Enable puppet agent
```bash
$ sudo puppet agent --enable
```

##### Sign certificate
```bash
$ sudo puppet agent -t
# Apply following two commands on the puppet master node
puppet $ sudo puppet cert list
puppet $ sudo puppet cert sign compute<N>.openstacklocal
```

##### Dirty Hacks
Facter currently uses `ifconfig` to grab local ip addresses. However, `ifconfig`
cannot correctly identify multiple addresses bound to a single interface, while
we have both management network and data network set to br100 in this exercise.
This hack make use of `ip` from `iproute2` to find the correct local address of
management network.

```bash
$ sudo sed -i 's|output = Facter::Util::IP.exec_ifconfig(\["2>/dev/null"\])|output = Facter::Core::Execution.exec("/sbin/ip addr 2>/dev/null")|g' /usr/lib/ruby/vendor_ruby/facter/ipaddress.rb
```

##### Initial run
```bash
$ sudo puppet agent -t
```
