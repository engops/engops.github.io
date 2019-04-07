---
layout: default
title: Ironic StandAlone
parent: Ironic
gran_parent: OpenStack
nav_order: 2
---

#  Instalation and using Ironic as standalone project
---

```
setenforce 0
sed s/SELINUX=.*/SELINUX=disabled/ /etc/selinux/config -i
```

```
systemctl stop firewalld
systemctl disable firewalld
```

interface de gerencia
```
cat << EOF > /etc/sysconfig/network-scripts/ifcfg-eth0 
TYPE=Ethernet
BOOTPROTO=static
NAME=eth0
DEVICE=eth0
ONBOOT=yes
IPADDR=172.25.49.58
PREFIX=22
GATEWAY=172.25.48.129
DNS1=8.8.8.8
EOF
```

interface do dhcp
```
cat << EOF > /etc/sysconfig/network-scripts/ifcfg-eth1
TYPE=Ethernet
BOOTPROTO=static
NAME=eth1
DEVICE=eth1
ONBOOT=yes
IPADDR=192.168.0.2
PREFIX=24
EOF
```

#### pacotes basicos e repositorios
```
yum install vim epel-release bash-completion centos-release-openstack-rocky -y
```

banco de dados para o ironic
```
yum install mariadb-server -y

systemctl enable mariadb.service
systemctl start mariadb.service

mysql -uroot -e "CREATE DATABASE ironicdb CHARACTER SET utf8"
mysql -uroot -e "GRANT ALL PRIVILEGES ON ironicdb.* TO 'ironicuser'@'localhost' IDENTIFIED BY 'ironicpass'"
mysql -uroot -e "GRANT ALL PRIVILEGES ON ironicdb.* TO 'ironicuser'@'%' IDENTIFIED BY'ironicpass'"
```

serviço de messageria
```
yum install rabbitmq-server.noarch -y

systemctl enable rabbitmq-server.service
systemctl start rabbitmq-server.service
```

serviço de dhcp
```
yum install dhcp -y
cat << EOF > /etc/dhcp/dhcpd.conf
authoritative;
allow unknown-clients;
allow booting;
allow bootp;
option ip-forwarding    false;
option mask-supplier    false;
subnet 192.168.0.0 netmask 255.255.255.0 {
      range 192.168.0.100 192.168.0.254;
      option routers                          192.168.0.1;
      option domain-name-servers              8.8.8.8;
      option subnet-mask                      255.255.255.0;
      next-server                     192.168.0.2;
      filename                        "pxelinux.0";
      max-lease-time                  86400;
      default-lease-time              43200;
      min-lease-time                  43200;
}
EOF

systemctl enable dhcpd
systemctl start dhcpd
```

serviço de tftp
```
yum install tftp-server syslinux-tftpboot xinetd tftp -y

mv /var/lib/tftpboot /
ln -s /tftpboot /var/lib/
mkdir -p /tftpboot/master_images
mkdir -p /tftpboot/pxelinux.cfg
mkdir -p /tftpboot/ironic

echo 're ^(/tftpboot/) /tftpboot/\2' > /tftpboot/map-file
echo 're ^/tftpboot/ /tftpboot/' >> /tftpboot/map-file
echo 're ^(^/) /tftpboot/\1' >> /tftpboot/map-file
echo 're ^([^/]) /tftpboot/\1' >> /tftpboot/map-file
```

```
systemctl enable xinetd.service tftp.service
systemctl start xinetd.service tftp.service
```

serivço de http
```
yum install httpd -y
mkdir /var/www/html/httpboot

systemctl enable httpd.service
systemctl start httpd.service
```

```
yum install qemu-img psmisc wget python-networkx squashfs-tools policycoreutils-python libguestfs-bash-completion libvirt diskimage-builder -y

systemctl enable libvirtd
systemctl start libvirtd

wget http://tarballs.openstack.org/ironic-python-agent/coreos/files/coreos_production_pxe.vmlinuz -O /tftpboot/ironic/coreos_production_pxe.vmlinuz
wget http://tarballs.openstack.org/ironic-python-agent/coreos/files/coreos_production_pxe_image-oem.cpio.gz -O /tftpboot/ironic/coreos_production_pxe_image-oem.cpio.gz

disk-image-create centos7 baremetal dhcp-all-interfaces grub2 -o centos7

export LIBGUESTFS_BACKEND=direct
virt-sysprep -a centos7.qcow2 --root-password password:12

mv centos7.initrd /var/www/html/httpboot
mv centos7.qcow2 /var/www/html/httpboot
mv centos7.vmlinuz /var/www/html/httpboot
```

pacotes do ironic e dos clientes
```
yum install openstack-ironic-conductor.noarch openstack-ironic-api.noarch python-ironicclient python-openstackclient -y

cat << EOF > /etc/ironic/ironic.conf
[DEFAULT]
auth_strategy = noauth
enabled_hardware_types = ipmi
enabled_bios_interfaces = fake,ilo,irmc,no-bios
enabled_console_interfaces = ipmitool-socat,no-console
default_console_interface = no-console
enabled_deploy_interfaces = iscsi,direct
default_deploy_interface = direct
enabled_management_interfaces = ipmitool,noop
enabled_network_interfaces = flat,noop
default_network_interface = noop
enabled_power_interfaces = ipmitool
enabled_vendor_interfaces = ipmitool,no-vendor
my_ip = 172.25.49.58
debug = true
log_dir = /var/log/ironic
transport_url = rabbit://guest:guest@localhost:5672/
[agent]
[ansible]
[api]
host_ip = 172.25.49.58
port = 6385
[audit]
[cimc]
[cinder]
[cisco_ucs]
[conductor]
auth_type = none
api_url = http://172.25.49.58:6385
sync_power_state_interval = 60
send_sensor_data = true
send_sensor_data_types = Temperature,Fan,Voltage
[console]
terminal_cert_dir = /tmp/ca
[cors]
[database]
connection=mysql+pymysql://ironicuser:ironicpass@localhost/ironicdb?charset=utf8
[deploy]
http_url = 172.25.49.58:80
http_root = /httpboot
[dhcp]
dhcp_provider = none
[disk_partitioner]
[disk_utils]
[drac]
[glance]
[healthcheck]
[ilo]
[inspector]
endpoint_override = http://172.25.49.58:6385
[ipmi]
[irmc]
[ironic_lib]
[iscsi]
[keystone_authtoken]
[matchmaker_redis]
[metrics]
[metrics_statsd]
[neutron]
[oneview]
[oslo_concurrency]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
driver = messaging
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_policy]
[profiler]
[pxe]
pxe_append_params=nofb nomodeset vga=normal console=tty0 console=ttyS0,115200n8
tftp_server = 172.25.49.58
tftp_root = /tftpboot
tftp_master_path = /tftpboot/master_images
pxe_bootfile_name = pxelinux.0
pxe_config_subdir = pxelinux.cfg
[service_catalog]
[snmp]
[ssl]
[swift]
[xclarity]
EOF

chown -R ironic: /tftpboot

ironic-dbsync --config-file /etc/ironic/ironic.conf create_schema

systemctl enable openstack-ironic-api openstack-ironic-conductor

systemctl start openstack-ironic-api openstack-ironic-conductor
```

```
cat << EOF > ~/RC_IRONIC
export OS_AUTH_TOKEN=fake-token
export IRONIC_URL=http://172.25.49.58:6385/
export OS_TOKEN=fake-token
export OS_URL=http://172.25.49.58:6385/
EOF

source ~/RC_IRONIC

ironic node-list

openstack baremetal node list
```


```
nodename=compute-teste
nodeuuid="1e1ba69d-a6df-4f4a-8653-fa0402c591d6"
macaddr="52:54:00:58:0c:fc"
portid="4609e71f-c7db-490d-ba75-b3c338129b24"
ipmiaddr="172.25.48.129"
ipmiuser="root"
ipmipass="calvin"
ipmiport="7701"
imagemd5=$(md5sum /var/www/html/httpboot/centos7.qcow2 | awk '{print $1}')
```

```
#### no service-node
vbmc add ${nodename} --port ${ipmiport} --address ${ipmiaddr} --username ${ipmiuser} --password ${ipmipass}
vbmc start ${nodename} 
####
```

```
openstack baremetal node create --driver ipmi --uuid ${nodeuuid} --name ${nodename} \
    --driver-info ipmi_address=${ipmiaddr} \
    --driver-info ipmi_username=${ipmiuser} \
    --driver-info ipmi_password=${ipmipass} \
    --driver-info ipmi_port=${ipmiport} \
    --driver-info deploy_kernel=file:///tftpboot/ironic/coreos_production_pxe.vmlinuz \
    --driver-info deploy_ramdisk=file:///tftpboot/ironic/coreos_production_pxe_image-oem.cpio.gz
```

```
openstack baremetal port create ${macaddr} --node ${nodeuuid}
```

```
openstack baremetal node set ${nodeuuid} \
    --instance-info image_source=http://172.25.49.58:80/httpboot/centos7.qcow2 \
    --instance-info image_checksum=${imagemd5} \
    --instance-info capabilities='{"boot_option": "local"}' \
    --instance-info kernel=http://172.25.49.58:80/httpboot/centos7.vmlinuz \
    --instance-info ramdisk=http://172.25.49.58:80/httpboot/centos7.initrd \
    --instance-info root_gb=10 \
    --management-interface noop \
    --bios-interface no-bios \
    --console-interface no-console \
    --inspect-interface no-inspect \
    --raid-interface no-raid \
    --rescue-interface no-rescue
```

```
openstack baremetal node validate ${nodeuuid}

openstack baremetal node manage ${nodeuuid}

openstack baremetal node provide ${nodeuuid}

openstack baremetal node deploy ${nodeuuid}
```
