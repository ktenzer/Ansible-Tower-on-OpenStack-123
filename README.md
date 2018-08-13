# General
The purpose of this project is to provide a simple, yet flexible deployment of OpenShift on OpenStack using a three step process. This guide assumes you are familiar with OpenStack.

# Contribution
If you want to provide additional features, please feel free to contribute via pull requests or any other means.
We are happy to track and discuss ideas, topics and requests via 'Issues'.

# Releases
For each release of Ansible Tower there will be a release branch with the Tower version created. This is to handle differences between versions of Tower. The master branch is always experimental and may not work.

# Pre-requisites
* Working OpenStack deployment. Tested is OpenStack 12 & 13 (Pike & Queens) using RDO.
* RHEL 7 image. Tested is RHEL 7.5.
* An openstack ssh key for accessing instances.
* A provider (public) network with at least two or three available floating ips.
* A service (internal) network
* A router configured with the public network as gateway and internal network as interface.
* Flavors configured for OpenShift. These are only recommendations.
  * tower.node  (2 vCPU, 4GB RAM, 30 GB Root Disk)
  * tower.db   (4 vCPU, 8GB RAM, 30 GB Root Disk)
* Properly configured cinder and nova storage.
  * Make sure you aren't using default loop back and have disabled disk zeroing in cinder/nova for LVM.

More information on setting up proper OpenStack environment can be found [here](https://keithtenzer.com/2018/07/17/openstack-13-queens-lab-installation-and-configuration-guide-for-hetzner-root-servers/).

# OpenStack Pre-Configuration (if required)

## Setup External Network
You need an external floating ip network, and internal network and a router using the external network as gateway and internal network interface. In this case we will create the public network and router in the admin project and the internal network in a new project.

Authenticate to admin project

```
# source /root/keystonerc_admin
```

Create public network

```
# openstack network create --provider-network-type flat \
--provider-physical-network extnet2 --external public
```

Create public subnet

```
# openstack subnet create --network public --allocation-pool \
start=144.76.132.226,end=144.76.132.230 --no-dhcp \
--subnet-range 144.76.132.224/29 public_subnet
```

Create router

```
# openstack router create --no-ha router1

```

Set public network as gateway in router

```
# openstack router set --external-gateway public router1
```

## Setup New OpenStack Project

Create Project

```
# openstack project create test
```

Add admin user as admin to project

```
# openstack role add --project test --user admin admin
```` 

Increase project quota for security groups

```
# openstack quota set --secgroups 100 test
```

Increase quota for volumes

```
#  openstack quota set --volumes 100 test
```

## Setup Internal Network

Create internal network

```
# openstack network create test
```

Create internal subnet

```
# openstack subnet create --network test --allocation-pool \
start=192.168.4.100,end=192.168.4.200 --dns-nameserver 213.133.98.98 \
--subnet-range 192.168.4.0/24 test_subnet
```

Add internal network to router as interface

```
# openstack router add subnet router1 test_subnet
```

# Installation
There are two supported configurations: single node and multi-node. In case of single node Tower scheduler, API/UI is run on towerdb node, so everything on towerdb. In case of multi-node the towerdb will run the database and tower scheduler/API/UI will run on separate nodes, the number is configurable.

Install Git & Ansible
```
# yum install -y git ansible
```

Clone Git Repository
```
# git clone https://github.com/ktenzer/ansible-tower-on-openstack-123.git
```

Change dir to repository
```
# cd ansible-tower-on-openstack-123
```

Checkout release branch 3.2.5
```
# git checkout release-3.2.5
```

Configure Parameters
```
# cp sample-vars.yml vars.yml
```
```
# vi vars.yml
---
### General Settings ###
towerdb_hostname: towerdb
tower_db_user: tower
tower_db_passwd: <tower db password>
tower_admin_passwd: <tower admin password>
debug: True

### OpenStack Setting ###
openstack_user: admin
openstack_passwd: <openstack admin password>
openstack_ip: <openstack API IP>
openstack_project: admin
domain_name: lab
dns_forwarders: [213.133.98.98, 213.133.98.99]
external_network: public
service_network: mgmt
service_subnet: mgmt_subnet
image: rhel75
ssh_user: cloud-user
ssh_key_path: /root/admin.pem
ssh_key_name: admin
stack_name: tower
openstack_version: 13
heat_template_path: /root/ansible-tower-on-openstack-123/heat/tower_single_node.yaml

# For multinode use following heat template
#heat_template_path: /root/ansible-tower-on-openstack-123/heat/tower.yaml

### Red Hat Subscription ###
rhn_username: <rhn user>
rhn_password: <rhn password>
rhn_pool: <rhn pooll id>

### Tower Instance Count ###
node_count: 0

### OpenStack Instance Group Policies ###
### Set to 'anti-affinity' if multiple compute nodes and anti-affinity desired ###
node_server_group_policies: "['affinity']"

### OpenStack Instance Flavors ###
node_flavor: tower.node
towerdb_flavor: tower.db
```

Disable host key checking
```
# export ANSIBLE_HOST_KEY_CHECKING=False
```


Deploy Ansible Tower on OpenStack
```
# ansible-playbook deploy-tower.yml --private-key=/root/admin.pem -e @vars.yml
PLAY RECAP ****************************************************************************************************************************************************************************************
localhost                  : ok=10   changed=4    unreachable=0    failed=0
towerdb                    : ok=33   changed=26   unreachable=0    failed=0
```
Login to the UI and add license
```
https://towerip
```
