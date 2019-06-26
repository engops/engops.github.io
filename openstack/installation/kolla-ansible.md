---
layout: default
title: Kolla Ansible
parent: Installation
gran_parent: OpenStack
nav_order: 2
---

#  OpenStack via Kolla Ansible

---

## Topology

* In this topology will be used 1 virtual machine to kollai node, 3 virtual controller nodes and 2 virtual compute nodes;
* Both connections APIs and tenants will trough the network 192.168.24.0/24 on eth0 interface;
* Public external network will trough eth1 interface using neutron openvswitch;
* Nova Compute, Glance, Cinder and Gnocchi on external Ceph;
* Manila external storage (Isilon);
* Designate external DNS (PowerDNS);

---

## Prepare the hosts

```
sed -e 's/SELINUX=.*/SELINUX=permissive/' /etc/selinux/config -i
setenforce 0

systemctl stop firewalld
systemctl disable firewalld

cat << EOF >> /etc/hosts
192.168.24.2    kollanode.localdomain         kollanode
192.168.24.3    controllernode00.localdomain  controllernode00
192.168.24.4    controllernode01.localdomain  controllernode01
192.168.24.5    controllernode02.localdomain  controllernode02
192.168.24.6    computenode00.localdomain     computenode00
192.168.24.7    computenode01.localdomain     computenode01
EOF

* Configure ssh public key authentication to access all nodes from the kolla node

```
---

## Kolla node

### Install
```
yum install -y epel-release
yum install -y python-pip
pip install -U pip
yum install -y python-devel libffi-devel gcc openssl-devel libselinux-python
pip install kolla-ansible
```

### Inventory
```
cp -r /usr/share/kolla-ansible/etc_examples/kolla /etc/kolla/
cp /usr/share/kolla-ansible/ansible/inventory/* .


* Add the hosts in the inventory file multinode as below:

...
[control]
controllernode00
controllernode01
controllernode02

[network]
controllernode00
controllernode01
controllernode02

[external-compute]
computenode00
computenode01

[monitoring]
controllernode00

[storage]
controllernode00
controllernode01
controllernode02
...

* verify if all nodes are reachable
ansible -i ~/multinode -m ping all
```

### Edit the globals.yml file as follow:
```
...
kolla_base_distro: "centos" 
kolla_install_type: "source"
openstack_release: "rocky"  
kolla_internal_vip_address: "192.168.24.254"
network_interface: "eth0"   
neutron_external_interface: "eth1"
neutron_plugin_agent: "openvswitch"
nova_console: "novnc"
enable_aodh: "yes"
enable_barbican: "yes"
enable_ceilometer: "yes"
enable_ceph: "no"
enable_chrony: "yes"
enable_cinder: "yes"
enable_cinder_backup: "no"
enable_designate: "yes"
enable_gnocchi: "yes"
enable_haproxy: "yes"
enable_heat: "yes"
enable_horizon: "yes"
enable_horizon_designate: "{{ enable_designate | bool }}"
enable_horizon_manila: "{{ enable_manila | bool }}"
enable_horizon_neutron_lbaas: "{{ enable_neutron_lbaas | bool }}"
enable_manila: "yes"
enable_neutron_lbaas: "yes"
enable_panko: "yes"
enable_redis: "yes"
keystone_token_provider: 'fernet'
keystone_admin_user: "admin"
keystone_admin_project: "admin"
glance_backend_ceph: "yes"
glance_backend_file: "no"
glance_enable_rolling_upgrade: "no"
gnocchi_backend_storage: "ceph"
cinder_backend_ceph: "yes"
designate_backend: "pdns4"
nova_backend_ceph: "yes"
...
```

### Generate random passwords
```
kolla-genpwd
```

### Openstak custom config
```
[root@kollanode ~]# tree /etc/kolla/config
/etc/kolla/config
├── chrony
│   └── chrony.conf
├── cinder
│   ├── ceph.client.cinder.keyring
│   ├── ceph.conf
│   ├── cinder-volume
│   │   └── ceph.client.cinder.keyring
│   └── cinder-volume.conf
├── designate
│   └── pools.yaml
├── glance
│   ├── ceph.client.glance.keyring
│   ├── ceph.conf
│   └── glance-api.conf
├── gnocchi
│   ├── ceph.client.gnocchi.keyring
│   ├── ceph.conf
│   └── gnocchi.conf
├── manila
│   ├── manila.conf
│   └── manila-share.conf
└── nova
    ├── ceph.client.cinder.keyring
    ├── ceph.client.nova.keyring
    ├── ceph.conf
    └── nova-compute.conf

8 directories, 18 files

mkdir -p /etc/kolla/config
mkdir -p /etc/kolla/config/{chrony,cinder,designate,glance,gnocchi,manila,nova}

cat << EOF > /etc/kolla/config/chrony/chrony.conf
server 192.168.24.2 iburst
EOF

cat << EOF > /etc/kolla/config/cinder/cinder-volume.conf
[DEFAULT]
enabled_backends=ceph

[ceph]
volume_driver = "cinder.volume.drivers.rbd.RBDDriver"
volume_backend_name = "ceph"
rbd_cluster_name = "ceph"
rbd_pool = "cinder"
rbd_ceph_conf = "/etc/ceph/ceph.conf"
rbd_flatten_volume_from_snapshot = "false"
rbd_max_clone_depth = "5"
rbd_store_chunk_size = "4"
rados_connect_timeout = "-1"
glance_api_version = "2"
rbd_user = "cinder"
rbd_secret_uuid = {{ rbd_secret_uuid }}
EOF

cat << EOF > /etc/kolla/config/designate/pools.yaml 
- also_notifies: []
  attributes: {}
  description: Default Pool
  id: {{ designate_pool_id }}
  name: default
  nameservers:
  - host: 192.168.24.3
    port: 53
  - host: 192.168.24.4
    port: 53
  - host: 192.168.24.5
    port: 53
  ns_records:
  - hostname: ns001.localdomain.
    priority: 1
  - hostname: ns002.localdomain.
    priority: 2
  - hostname: ns003.localdomain.
    priority: 3
  targets:
  - description: PowerDNS4 DNS Server
    masters:
    - host: 192.168.24.3
      port: 5354
    - host: 192.168.24.4
      port: 5354
    - host: 192.168.24.5
      port: 5354
    options:
      api_endpoint: http://192.168.24.3:8081
      api_token: de9ad2e4-d89c-467a-b165-73014794699e
      host: 192.168.24.3
      port: '53'
    type: pdns4
EOF

* change the NS entry, endpoint ip address and token according with you PowerDNS setup.
* If you don't have a PowerDNS installed, just ignore the config above and set designate_backend to bind9 in the global config file.



cat << EOF > /etc/kolla/config/glance/glance-api.conf
[glance_store]
stores = rbd
default_store = rbd
rbd_store_pool = glance
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_chunk_size = 4
EOF


cat << EOF > /etc/kolla/config/gnocchi/gnocchi.conf
[storage]
driver = ceph
ceph_pool = gnocchi
ceph_username = gnocchi
ceph_keyring = /etc/ceph/ceph.client.gnocchi.keyring
ceph_conffile = /etc/ceph/ceph.conf
EOF


cat << EOF > /etc/kolla/config/manila/manila.conf 
[DEFAULT]
default_share_type = default_share_type
share_name_template = share-%s
EOF

cat << EOF > /etc/kolla/config/manila/manila-share.conf
[DEFAULT]
default_share_type = default_share_type
enabled_share_backends = isilon
enabled_share_protocols = NFS,CIFS
share_name_template = share-%s

[isilon]
share_backend_name = ISILON
share_driver = manila.share.drivers.dell_emc.driver.EMCShareDriver
driver_handles_share_servers = False
emc_share_backend = isilon
emc_nas_server = << ISILON IP ADDRESS >>
emc_nas_server_port = 8080
emc_nas_login = << ISIOLON USER >>
emc_nas_password = << ISILON PASS >>
emc_nas_root_dir = << ISILON ROOT DIR WHERE THE SHARES WILL BE CREATED >>
EOF


cat << EOF > /etc/kolla/config/nova/nova-compute.conf
[libvirt]
virt_type=qemu
images_type = rbd
images_rbd_pool = nova
images_rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = nova
rbd_secret_uuid = {{ rbd_secret_uuid }}
inject_password = false
inject_key = false
inject_partition = -2
live_migration_flag = "VIR_MIGRATE_UNDEFINE_SOURCE,VIR_MIGRATE_PEER2PEER,VIR_MIGRATE_LIVE,VIR_MIGRATE_PERSIST_DEST,VIR_MIGRATE_TUNNELLED"
disk_cachemodes = "none"
EOF

```

### Deploy
```
kolla-ansible -i ~/multinode bootstrap-servers

kolla-ansible -i ~/multinode prechecks

kolla-ansible -i ~/multinode deploy

* if somenthing goes wrong with mariadb bootstrap, maybe it can help:
kolla-ansible -i ~/multinode mariadb_recovery
```
---
That's it =)
