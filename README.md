Cloud System Administration Exercise
====================================

Exercise One
------------

### Notice

This is a quick guide for building a minimal OpenStack multi-node environment.
Always refer to http://docs.openstack.org/ for more details.

### Overview

An example set up of two-node architecture

* Controller Node
    - Identity (Keystone)
    - Compute Management (Nova)
    - Networking Management (Nova Network)
    - Image Service (Glance)
    - Dashboard (Horizon)
* Compute Node
    - Compute (Nova)
    - Networking (Nova Network)

### Prerequisite

Deploy three Ubuntu 14.04 LTS (Trust Tahr) nodes on
https://openstack-grizzly.nctu.edu.tw, one controller node and two compute nodes.

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
