---
layout: default
title: From Scratch - HA
parent: Installation
gran_parent: OpenStack
nav_order: 1
---

# OpenStack Queens from scratch on CentOS

---

## Topology

* In this topology will be used 3 controller nodes with pacemaker/corosync managed;
* Both connections APIs and tenants will trough the network 192.168.24.0/24 on eth0 interface;
* Public external network will trough eth1 interface using neutron linuxbridge;
* Nova Compute, Glance, Cinder on external Ceph;

---

## Prepare the hosts

```
sed -e 's/SELINUX=.*/SELINUX=permissive/' /etc/selinux/config -i
setenforce 0

systemctl stop firewalld
systemctl disable firewalld

cat << EOF >> /etc/sysctl.conf
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward=1
EOF
sysctl -p

cat << EOF >> /etc/hosts
192.168.24.2    controllernode00.localdomain  controllernode00
192.168.24.3    controllernode01.localdomain  controllernode01
192.168.24.4    controllernode02.localdomain  controllernode02
192.168.24.254  api.localdomain               api
192.168.24.254  db.localdomain                db
EOF


* enable CentOS extra repo if it isn't already enabled to install openstack repo

yum install -y centos-release-openstack-queens yum-utils yum-priorities
yum-config-manager --enable centos-openstack-queens --setopt="centos-openstack-queens.priority=10"
yum install -y python-openstackclient openstack-selinux
yum update -y && yum upgrade -y
reboot
```
---

## NTP config


#### Controller nodes

```
yum -y install chrony

cat << EOF > /etc/chrony.conf
server a.ntp.br iburst
server b.ntp.br iburst
server c.ntp.br iburst
allow 192.168.24.0/24
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
EOF

systemctl enable chronyd.service
systemctl start chronyd.service

chronyc sources
```

#### Compute nodes

```
yum install -y chrony

cat <<EOF >  /etc/chrony.conf
server controllernode00.localdomain iburst
server controllernode01.localdomain iburst
server controllernode02.localdomain iburst
EOF

systemctl enable chronyd.service
systemctl start chronyd.service

chronyc sources
```
---

## Configure pacemaker/corosync on controller nodes


#### Pacemaker cluster setup

```
yum install -y pacemaker pcs corosync

systemctl enable pcsd pacemaker corosync
systemctl stop pacemaker corosync
systemctl restart pcsd

echo CHANGEME | passwd --stdin hacluster


* from controllernode00
pcs cluster auth controllernode00 controllernode01 controllernode02 -u hacluster -p CHANGEME --force
pcs cluster setup --force --name openstack controllernode00 controllernode01 controllernode02
pcs cluster start --all

pcs property set stonith-enabled=false
```


#### VIP create
```
pcs resource create vip ocf:heartbeat:IPaddr2 cidr_netmask=24 ip=192.168.24.254 op monitor interval=30s on-fail=restart start interval=0s on-fail=restart stop interval=0s timeout=20s
```
---

## Configure OpenStack infra services

### Haproxy

```
yum install -y haproxy

cat << EOF > /etc/haproxy/haproxy.cfg
global
 log /dev/log  local0
 chroot /var/lib/haproxy
 user haproxy
 group haproxy
 daemon
 stats socket /var/lib/haproxy/stats

defaults
 log global
 mode tcp
 maxconn 300000
 option dontlognull
 option redispatch
 timeout connect 30s
 timeout client 70m
 timeout server 70m
 timeout check 10s
 timeout queue 2m
 timeout http-request 30s

## status of haproxy
listen stats
       bind 0.0.0.0:81
       mode http
       stats enable
       stats realm "HAProxy Statistics"
       stats uri /
       stats auth admin:123456

listen dashboard_cluster
 bind 192.168.24.254:80
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server controllernode00 controllernode00.localdomain:80 check weight 1
 server controllernode01 controllernode01.localdomain:80 check weight 1
 server controllernode02 controllernode02.localdomain:80 check weight 1

listen galera_cluster
 bind 192.168.24.254:3306
 balance  source
 option  mysql-check
 server controllernode00 controllernode00.localdomain:3306 check inter 2000 rise 2 fall 5
 server controllernode01 controllernode01.localdomain:3306 backup check inter 2000 rise 2 fall 5
 server controllernode02 controllernode02.localdomain:3306 backup check inter 2000 rise 2 fall 5

listen glance_api_cluster
 bind 192.168.24.254:9292
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server controllernode00 controllernode00.localdomain:9292 check weight 1
 server controllernode01 controllernode01.localdomain:9292 check weight 1
 server controllernode02 controllernode02.localdomain:9292 check weight 1

listen glance_registry_cluster
 bind 192.168.24.254:9191
 balance  source
 option  tcpka
 option  tcplog
 server controllernode00 controllernode00.localdomain:9191 check weight 1
 server controllernode01 controllernode01.localdomain:9191 check weight 1
 server controllernode02 controllernode02.localdomain:9191 check weight 1

listen keystone_admin_cluster
 bind 192.168.24.254:35357
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server controllernode00 controllernode00.localdomain:35357 check weight 1
 server controllernode01 controllernode01.localdomain:35357 check weight 1
 server controllernode02 controllernode02.localdomain:35357 check weight 1

listen keystone_public_internal_cluster
 bind 192.168.24.254:5000
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server controllernode00 controllernode00.localdomain:5000 check weight 1
 server controllernode01 controllernode01.localdomain:5000 check weight 1
 server controllernode02 controllernode02.localdomain:5000 check weight 1

listen nova_compute_api_cluster
 bind 192.168.24.254:8774
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server controllernode00 controllernode00.localdomain:8774 check weight 1
 server controllernode01 controllernode01.localdomain:8774 check weight 1
 server controllernode02 controllernode02.localdomain:8774 check weight 1

listen nova_metadata_api_cluster
 bind 192.168.24.254:8775
 balance  source
 option  tcpka
 option  tcplog
 server controllernode00 controllernode00.localdomain:8775 check weight 1
 server controllernode01 controllernode01.localdomain:8775 check weight 1
 server controllernode02 controllernode02.localdomain:8775 check weight 1

 listen nova_placement_api
  bind 192.168.24.254:8778
  balance  source
  option  tcpka
  option  tcplog
  server controllernode00 controllernode00.localdomain:8778 check inter 2000 rise 2 fall 5
  server controllernode01 controllernode01.localdomain:8778 check inter 2000 rise 2 fall 5
  server controllernode02 controllernode02.localdomain:8778 check inter 2000 rise 2 fall 5

listen cinder_api_cluster
 bind 192.168.24.254:8776
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server controllernode00 controllernode00.localdomain:8776 check weight 1
 server controllernode01 controllernode01.localdomain:8776 check weight 1
 server controllernode02 controllernode02.localdomain:8776 check weight 1

listen nova_vncproxy_cluster
 bind 192.168.24.254:6080
 balance  source
 option  tcpka
 option  tcplog
 server controllernode00 controllernode00.localdomain:6080 check weight 1
 server controllernode01 controllernode01.localdomain:6080 check weight 1
 server controllernode02 controllernode02.localdomain:6080 check weight 1

listen neutron_api_cluster
 bind 192.168.24.254:9696
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server controllernode00 controllernode00.localdomain:9696 check weight 1
 server controllernode01 controllernode01.localdomain:9696 check weight 1
 server controllernode02 controllernode02.localdomain:9696 check weight 1

listen heat_cloudformation
 bind 192.168.24.254:8000
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server controllernode00 controllernode00.localdomain:8000 check weight 1
 server controllernode01 controllernode01.localdomain:8000 check weight 1
 server controllernode02 controllernode02.localdomain:8000 check weight 1

listen heat_orchestration
 bind 192.168.24.254:8004
 balance  source
 option  tcpka
 option  httpchk
 option  tcplog
 server controllernode00 controllernode00.localdomain:8004 check weight 1
 server controllernode01 controllernode01.localdomain:8004 check weight 1
 server controllernode02 controllernode02.localdomain:8004 check weight 1
EOF
```
#### haproxy resource create
```
pcs resource create haproxy systemd:haproxy op monitor interval=30s on-fail=restart start interval=0s on-fail=restart stop interval=0s timeout=100 meta failure-timeout=60s migration-threshold=2 target-role=Started

pcs constraint order vip then haproxy id=order-vip-haproxy-mandatory

pcs constraint colocation add vip with haproxy id=colocation-vip-haproxy-INFINITY
```

### Database

```
yum install -y mariadb-galera-server mariadb MySQL-python galera rsync xinetd


systemctl start mariadb
systemctl stop mariadb


cat << EOF > /etc/my.cnf.d/galera.cnf
[client]
port = 3306
socket = /var/lib/mysql/mysql.sock

[isamchk]
key_buffer_size = 16M

[mysqld]
pid-file = /var/run/mariadb/mariadb.pid
socket = /var/lib/mysql/mysql.sock
basedir = /usr
bind-address = 192.168.24.2
binlog_format = ROW
datadir = /var/lib/mysql
default-storage-engine = innodb
expire_logs_days = 10

key_buffer_size = 256M
max_allowed_packet = 512M
query_cache_size = 0
query_cache_type = 0
join_buffer_size = 256K
table_open_cache      = 524288
max_heap_table_size   = 50M
tmp_table_size        = 50M
open_files_limit = 65535


innodb_autoinc_lock_mode = 2
innodb_locks_unsafe_for_binlog = 1
max_binlog_size = 100M
max_connections = 65535 # avoid too many connections
port = 3306
skip-external-locking
skip-name-resolve = 1
ssl = false

thread_cache_size = 8
thread_stack = 256K
tmpdir = /tmp
user = mysql
wsrep_auto_increment_control = 1
wsrep_causal_reads = 0
wsrep_certify_nonPK = 1
wsrep_cluster_address="gcomm://controllernode00.localdomain,controllernode01.localdomain,controllernode02.localdomain"
wsrep_cluster_name = galera_openstack
wsrep_convert_LOCK_to_trx = 0
wsrep_debug = 0
wsrep_drupal_282555_workaround = 0
wsrep_max_ws_rows = 131072
wsrep_max_ws_size = 1073741824
wsrep_on = ON
wsrep_provider = /usr/lib64/galera/libgalera_smm.so
wsrep_provider_options = "gmcast.listen_addr=tcp://192.168.24.2:4567"
wsrep_retry_autocommit = 1
wsrep_slave_threads = 8
wsrep_sst_method = rsync

[mysqld_safe]
log-error = /var/log/mariadb/mariadb.log
nice = 0
socket = /var/lib/mysql/mysql.sock

[mysqldump]
max_allowed_packet = 16M
quick
EOF


sed -i 's/192.168.24.2/192.168.24.3/g' /etc/my.cnf.d/galera.cnf
sed -i 's/192.168.24.2/192.168.24.4/g' /etc/my.cnf.d/galera.cnf

cat << EOF > /etc/sysconfig/clustercheck
MYSQL_USERNAME=root
MYSQL_PASSWORD=CHANGEME
MYSQL_HOST=localhost
EOF

chmod 0600 /etc/sysconfig/clustercheck

cat << EOF > /etc/xinetd.d/galera-monitor
service galera-monitor
{
        port            = 9200
        bind            = 0.0.0.0
        disable         = no
        socket_type     = stream
        protocol        = tcp
        wait            = no
        user            = root
        group           = root
        groups          = yes
        server          = /usr/bin/clustercheck
        type            = UNLISTED
        per_source      = UNLIMITED
        log_on_success +=
        log_on_failure += HOST
        flags           = REUSE
        instances       = UNLIMITED
        log_on_success  =
}
EOF


systemctl enable xinetd
systemctl restart xinetd

/usr/libexec/mysqld --wsrep-new-cluster &

mysqladmin password 'CHANGEME'

mysql -u root -p'CHANGEME' -e "GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'CHANGEME' WITH GRANT OPTION; FLUSH PRIVILEGES"

mysql -u root -p'CHANGEME' -e "CREATE USER 'haproxy_check'@'%';"


killall mysqld
```

#### MariaDB Galera resource create

```
pcs resource create galera 'ocf:heartbeat:galera' enable_creation=true wsrep_cluster_address="gcomm://controllernode00.localdomain,controllernode01.localdomain,controllernode02.localdomain" additional_parameters=--open-files-limit=16384 meta master-max=3 ordered=true op promote timeout=300s on-fail=block master
```

### RabbitMQ

```
yum -y install rabbitmq-server

systemctl start rabbitmq-server
rabbitmqctl stop_app
rabbitmqctl reset
systemctl stop rabbitmq-server

* copy erlang cookie from controllernode00 to both other controller nodes
scp /var/lib/rabbitmq/.erlang.cookie  root@controllernode01:/var/lib/rabbitmq/
scp /var/lib/rabbitmq/.erlang.cookie root@controllernode02:/var/lib/rabbitmq/

echo '[rabbitmq_management].' > /etc/rabbitmq/enabled_plugins

chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie

chmod 400 /var/lib/rabbitmq/.erlang.cookie


* from both controllernode01 and controllernode02
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@controllernode00
```
#### rabbitmq resource create

```
pcs resource create rabbitmq-server  systemd:rabbitmq-server op monitor start-delay=20s meta ordered=true interleave=true --clone
```
#### rabbitmq user create

```
rabbitmqctl add_user rabbit-user CHANGEME
rabbitmqctl set_permissions rabbit-user ".*" ".*" ".*"
rabbitmqctl set_policy ha-all '^(?!amq\.).*' '{"ha-mode": "all"}'
```

### MemcacheD

```
yum install -y memcached


cat << EOF > /etc/sysconfig/memcached
PORT="11211"
USER="memcached"
MAXCONN="8192"
CACHESIZE="1024"
OPTIONS="-l 192.168.24.2" << CHANGE ME
EOF
```
#### memcached resource create

```
pcs resource create memcached systemd:memcached meta interleave=true op monitor start-delay=20s --clone
```
---

## Configure OpenStack services

### Keystone

```
mysql -u root -p'CHANGEME' -e "CREATE DATABASE keystone;"
mysql -u root -p'CHANGEME' -e "GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'CHANGEME'; FLUSH PRIVILEGES"
mysql -u root -p'CHANGEME' -e "GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'CHANGEME'; FLUSH PRIVILEGES"


yum install -y openstack-keystone httpd mod_wsgi


cat << EOF > /etc/keystone/keystone.conf
[DEFAULT]
[application_credential]
[assignment]
[auth]
[cache]
backend = oslo_cache.memcache_pool
enabled=true
memcache_servers = controllernode00.localdomain:11211,controllernode01.localdomain:11211,controllernode02.localdomain:11211
[catalog]
[cors]
[credential]
[database]
connection = mysql+pymysql://keystone:CHANGEME@db.localdomain/keystone
[domain_config]
[endpoint_filter]
[endpoint_policy]
[eventlet_server]
[federation]
[fernet_tokens]
[healthcheck]
[identity]
[identity_mapping]
[ldap]
[matchmaker_redis]
[memcache]
servers = controllernode00.localdomain:11211,controllernode01.localdomain:11211,controllernode02.localdomain:11211
[oauth1]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[paste_deploy]
[policy]
[profiler]
[resource]
[revoke]
[role]
[saml]
[security_compliance]
[shadow_users]
[signing]
[token]
provider = fernet
[tokenless_auth]
[trust]
[unified_limit]
EOF


sed -i 's/^#ServerName www.example.com:80/ServerName controllernode00/' /etc/httpd/conf/httpd.conf
sed -i 's/^#ServerName www.example.com:80/ServerName controllernode01/' /etc/httpd/conf/httpd.conf
sed -i 's/^#ServerName www.example.com:80/ServerName controllernode02/' /etc/httpd/conf/httpd.conf


sed -i 's/^Listen 80/Listen 192.168.24.2:80/' /etc/httpd/conf/httpd.conf
sed -i 's/^Listen 80/Listen 192.168.24.3:80/' /etc/httpd/conf/httpd.conf
sed -i 's/^Listen 80/Listen 192.168.24.4:80/' /etc/httpd/conf/httpd.conf


cat << EOF > /etc/httpd/conf.d/wsgi-keystone.conf
Listen 192.168.24.2:5000
Listen 192.168.24.2:35357


<VirtualHost *:5000>
    WSGIDaemonProcess keystone-public processes=15 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-public
    WSGIScriptAlias / /usr/bin/keystone-wsgi-public
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    LimitRequestBody 114688
    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    ErrorLog /var/log/httpd/keystone.log
    CustomLog /var/log/httpd/keystone_access.log combined


    <Directory /usr/bin>
        <IfVersion >= 2.4>
            Require all granted
        </IfVersion>
        <IfVersion < 2.4>
            Order allow,deny
            Allow from all
        </IfVersion>
    </Directory>
</VirtualHost>


<VirtualHost *:35357>
    WSGIDaemonProcess keystone-admin processes=15 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-admin
    WSGIScriptAlias / /usr/bin/keystone-wsgi-admin
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    LimitRequestBody 114688
    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    ErrorLog /var/log/httpd/keystone.log
    CustomLog /var/log/httpd/keystone_access.log combined


    <Directory /usr/bin>
        <IfVersion >= 2.4>
            Require all granted
        </IfVersion>
        <IfVersion < 2.4>
            Order allow,deny
            Allow from all
        </IfVersion>
    </Directory>
</VirtualHost>


Alias /identity /usr/bin/keystone-wsgi-public
<Location /identity>
    SetHandler wsgi-script
    Options +ExecCGI


    WSGIProcessGroup keystone-public
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
</Location>


Alias /identity_admin /usr/bin/keystone-wsgi-admin
<Location /identity_admin>
    SetHandler wsgi-script
    Options +ExecCGI


    WSGIProcessGroup keystone-admin
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
</Location>
EOF


sed -i 's/192.168.24.2/192.168.24.3/g' /etc/httpd/conf.d/wsgi-keystone.conf
sed -i 's/192.168.24.2/192.168.24.4/g' /etc/httpd/conf.d/wsgi-keystone.conf


su -s /bin/sh -c "keystone-manage db_sync" keystone


keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone


rsync -av /etc/keystone/fernet-keys controllernode01:/etc/keystone
rsync -av /etc/keystone/fernet-keys controllernode02:/etc/keystone
rsync -av /etc/keystone/credential-keys controllernode01:/etc/keystone
rsync -av /etc/keystone/credential-keys controllernode02:/etc/keystone


keystone-manage bootstrap --bootstrap-password CHANGEME        \
--bootstrap-admin-url http://api.localdomain:5000/v3/          \
--bootstrap-internal-url http://api.localdomain:5000/v3/       \
--bootstrap-public-url http://api.localdomain:5000/v3/         \
--bootstrap-region-id RegionOne


systemctl disable httpd.service
systemctl stop httpd.service
```

#### keystone resource create

```
pcs resource create httpd systemd:httpd meta interleave=true op monitor start-delay=10s --clone

pcs constraint order haproxy then httpd-clone id=order-haproxy-httpd-clone-mandatory
pcs constraint order memcached-clone then httpd-clone id=order-memcached-clone-httpd-clone-mandatory
pcs constraint order rabbitmq-server-clone then httpd-clone id=order-rabbitmq-server-clone-httpd-clone-mandatory
pcs constraint order promote galera-master then start openstack-keystone-clone id=order-galera-master-openstack-keystone-clone-mandatory
```

#### keystone post configure

```
cat << EOF > admin-openrc
#
export OS_USERNAME=admin
export OS_PASSWORD=CHANGEME
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://api.localdomain:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_AUTH_TYPE=password
EOF

source admin-openrc

openstack project create --domain default --description "Service Project" service
```

### Glance

```
mysql -u root -p'CHANGEME' -e "CREATE DATABASE glance;"
mysql -u root -p'CHANGEME' -e "GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'CHANGEME'; FLUSH PRIVILEGES"
mysql -u root -p'CHANGEME' -e "GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'CHANGEME'; FLUSH PRIVILEGES"


openstack user create --domain default --password CHANGEME glance
openstack role add --project service --user glance admin
openstack service create --name glance --description "OpenStack Image" image


openstack endpoint create --region RegionOne image public http://api.localdomain:9292
openstack endpoint create --region RegionOne image internal http://api.localdomain:9292
openstack endpoint create --region RegionOne image admin http://api.localdomain:9292

yum install -y openstack-glance


cat << EOF > /etc/glance/glance-api.conf
[DEFAULT]
enable_v1_api = false
enable_v2_api = true
bind_host=192.168.24.2
show_image_direct_url = True
show_multiple_locations = True
[database]
connection = mysql+pymysql://glance:CHANGEME@db.localdomain/glance
backend = sqlalchemy
[glance_store]
stores = rbd
default_store = rbd
rbd_store_chunk_size = 4
rbd_store_pool = glance
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
filesystem_store_datadir = /var/lib/glance/images/
[image_format]
disk_formats = ami,ari,aki,vhd,vhdx,vmdk,raw,qcow2,vdi,iso,ploop.root-tar
[keystone_authtoken]
auth_uri = http://api.localdomain:5000
auth_url = http://api.localdomain:35357
memcached_servers = controllernode00.localdomain:11211,controllernode01.localdomain:11211,controllernode02.localdomain:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = CHANGEME
[paste_deploy]
flavor = keystone
EOF


sed -i 's/192.168.24.2/192.168.24.3/' /etc/glance/glance-api.conf
sed -i 's/192.168.24.2/192.168.24.4/' /etc/glance/glance-api.conf


cat << EOF > /etc/glance/glance-registry.conf
[DEFAULT]
bind_host = 192.168.24.2
[database]
connection = mysql+pymysql://glance:CHANGEME@db.localdomain/glance
backend = sqlalchemy
[keystone_authtoken]
auth_uri = http://api.localdomain:5000
auth_url = http://api.localdomain:35357
memcached_servers = controllernode00.localdomain:11211,controllernode01.localdomain:11211,controllernode02.localdomain:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = CHANGEME
[paste_deploy]
flavor = keystone
EOF


sed -i 's/192.168.24.2/192.168.24.3/' /etc/glance/glance-registry.conf
sed -i 's/192.168.24.2/192.168.24.4/' /etc/glance/glance-registry.conf


su -s /bin/sh -c "glance-manage db_sync" glance


systemctl disable openstack-glance-api.service openstack-glance-registry.service
systemctl stop openstack-glance-api.service openstack-glance-registry.service
```

#### glance resource create

```
pcs resource create openstack-glance-registry systemd:openstack-glance-registry meta interleave=true op monitor start-delay=60s --clone
pcs resource create openstack-glance-api systemd:openstack-glance-api meta interleave=true op monitor start-delay=60s --clone

pcs constraint order httpd-clone then openstack-glance-registry-clone id=order-httpd-clone-openstack-glance-registry-clone-mandatory
pcs constraint order openstack-glance-registry-clone then openstack-glance-api-clone id=order-openstack-glance-registry-clone-openstack-glance-api-clone-mandatory

pcs constraint colocation add openstack-glance-api-clone with openstack-glance-registry-clone id=colocation-openstack-glance-api-clone-openstack-glance-registry-clone-INFINITY rsc-role=Started with-rsc-role=Master
```

#### glance post configure
```
wget http://download.cirros-cloud.net/0.3.6/cirros-0.3.6-x86_64-disk.img

openstack image create "cirros" --file cirros-0.3.6-x86_64-disk.img --disk-format qcow2 --container-format bare --public
```

### Nova

```
mysql -u root -p'CHANGEME' -e "CREATE DATABASE nova;"
mysql -u root -p'CHANGEME' -e "CREATE DATABASE nova_api;"
mysql -u root -p'CHANGEME' -e "CREATE DATABASE nova_cell0;"

mysql -u root -p'CHANGEME' -e "GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'CHANGEME'; FLUSH PRIVILEGES"
mysql -u root -p'CHANGEME' -e "GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'CHANGEME'; FLUSH PRIVILEGES"

mysql -u root -p'CHANGEME' -e "GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'CHANGEME'; FLUSH PRIVILEGES"
mysql -u root -p'CHANGEME' -e "GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'CHANGEME'; FLUSH PRIVILEGES"

mysql -u root -p'CHANGEME' -e "GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'CHANGEME'; FLUSH PRIVILEGES"
mysql -u root -p'CHANGEME' -e "GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'CHANGEME'; FLUSH PRIVILEGES"


openstack user create --domain default --password CHANGEME nova
openstack role add --project service --user nova admin
openstack service create --name nova --description "OpenStack Compute" compute


openstack user create --domain default --password CHANGEME placement
openstack role add --project service --user placement admin
openstack service create --name placement --description "Placement API" placement


openstack endpoint create --region RegionOne compute public http://api.localdomain:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://api.localdomain:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://api.localdomain:8774/v2.1

openstack endpoint create --region RegionOne placement public http://api.localdomain:8778
openstack endpoint create --region RegionOne placement internal http://api.localdomain:8778
openstack endpoint create --region RegionOne placement admin http://api.localdomain:8778

yum install -y openstack-nova-api openstack-nova-conductor \
  openstack-nova-console openstack-nova-novncproxy \
  openstack-nova-scheduler openstack-nova-placement-api


cat << EOF > /etc/nova/nova.conf
[DEFAULT]
transport_url = rabbit://rabbit-user:CHANGEME@controllernode00.localdomain:5672,rabbit-user:CHANGEME@controllernode01.localdomain:5672,rabbit-user:CHANGEME@controllernode02.localdomain:5672/
my_ip = 192.168.24.2
osapi_compute_listen=192.168.24.2
metadata_listen=192.168.24.2
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
state_path=/var/lib/nova
enabled_apis=osapi_compute,metadata
firewall_driver = nova.virt.firewall.NoopFirewallDriver
default_ephemeral_format = ext4
[api]
auth_strategy = keystone
[api_database]
connection = mysql+pymysql://nova:CHANGEME@db.localdomain/nova_api
[cache]
backend = oslo_cache.memcache_pool
enabled=true
memcache_servers = controllernode00.localdomain:11211,controllernode01.localdomain:11211,controllernode02.localdomain:11211
[cells]
#enable=false
[database]
connection = mysql+pymysql://nova:CHANGEME@db.localdomain/nova
[glance]
api_servers = http://api.localdomain:9292
[keystone_authtoken]
auth_uri = http://api.localdomain:5000
auth_url = http://api.localdomain:35357
memcached_servers = controllernode00.localdomain:11211,controllernode01.localdomain:11211,controllernode02.localdomain:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = CHANGEME
[neutron]
url = http://api.localdomain:9696
auth_url = http://api.localdomain:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = CHANGEME
service_metadata_proxy = true
metadata_proxy_shared_secret = METADATA_SECRET
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://api.localdomain:35357/v3
username = placement
password = CHANGEME
[vnc]
enabled = true
vncserver_listen = \$my_ip
novncproxy_host = 192.168.24.2
vncserver_proxyclient_address = \$my_ip
novncproxy_base_url = http://api.localdomain:6080/vnc_auto.html
[wsgi]
api_paste_config=/etc/nova/api-paste.ini
[scheduler]
discover_hosts_in_cells_interval = 300
EOF


sed -i 's/192.168.24.2/192.168.24.3/g' /etc/nova/nova.conf
sed -i 's/192.168.24.2/192.168.24.4/g' /etc/nova/nova.conf

cat << EOF > /etc/httpd/conf.d/00-nova-placement-api.conf
Listen 192.168.24.2:8778


<VirtualHost *:8778>
  WSGIProcessGroup nova-placement-api
  WSGIApplicationGroup %{GLOBAL}
  WSGIPassAuthorization On
  WSGIDaemonProcess nova-placement-api processes=30 threads=20 user=nova group=nova
  WSGIScriptAlias / /usr/bin/nova-placement-api
  <IfVersion >= 2.4>
    ErrorLogFormat "%M"
  </IfVersion>
  ErrorLog /var/log/nova/nova-placement-api.log

<Directory /usr/bin>
    <IfVersion >= 2.4>
        Require all granted
    </IfVersion>
    <IfVersion < 2.4>
        Order allow,deny
        Allow from all
    </IfVersion>
</Directory>
</VirtualHost>


Alias /nova-placement-api /usr/bin/nova-placement-api
<Location /nova-placement-api>
  SetHandler wsgi-script
  Options +ExecCGI
  WSGIProcessGroup nova-placement-api
  WSGIApplicationGroup %{GLOBAL}
  WSGIPassAuthorization On
 Require all granted  
</Location>
EOF


sed -i 's/192.168.24.2/192.168.24.3/g' /etc/httpd/conf.d/00-nova-placement-api.conf
sed -i 's/192.168.24.2/192.168.24.4/g' /etc/httpd/conf.d/00-nova-placement-api.conf


systemctl restart httpd


su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
su -s /bin/sh -c "nova-manage db sync" nova


nova-manage cell_v2 list_cells


systemctl disable openstack-nova-api.service \
  openstack-nova-consoleauth.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service

systemctl stop openstack-nova-api.service \
  openstack-nova-consoleauth.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service
```

#### nova resource create
```
pcs resource create openstack-nova-api systemd:openstack-nova-api meta interleave=true op monitor start-delay=10s --clone
pcs resource create openstack-nova-consoleauth systemd:openstack-nova-consoleauth meta interleave=true op monitor start-delay=10s --clone
pcs resource create openstack-nova-scheduler systemd:openstack-nova-scheduler meta interleave=true op monitor start-delay=10s --clone
pcs resource create openstack-nova-conductor systemd:openstack-nova-conductor meta interleave=true op monitor start-delay=10s --clone
pcs resource create openstack-nova-novncproxy systemd:openstack-nova-novncproxy meta interleave=true op monitor start-delay=10s --clone


pcs constraint order openstack-nova-api-clone then openstack-nova-scheduler-clone id=order-openstack-nova-api-clone-openstack-nova-scheduler-clone-mandatory
pcs constraint order openstack-nova-novncproxy-clone then openstack-nova-api-clone id=order-openstack-nova-novncproxy-clone-openstack-nova-api-clone-mandatory
pcs constraint order openstack-nova-consoleauth-clone then openstack-nova-novncproxy-clone id=order-openstack-nova-consoleauth-clone-openstack-nova-novncproxy-clone-mandatory
pcs constraint order openstack-nova-scheduler-clone then openstack-nova-conductor-clone id=order-openstack-nova-scheduler-clone-openstack-nova-conductor-clone-mandatory
pcs constraint order httpd-clone then openstack-nova-consoleauth-clone id=order-httpd-clone-openstack-nova-consoleauth-clone-mandatory


pcs constraint colocation add openstack-nova-api-clone with openstack-nova-novncproxy-clone id=colocation-openstack-nova-api-clone-openstack-nova-novncproxy-clone-INFINITY rsc-role=Started with-rsc-role=Master
pcs constraint colocation add openstack-nova-scheduler-clone with openstack-nova-api-clone id=colocation-openstack-nova-scheduler-clone-openstack-nova-api-clone-INFINITY rsc-role=Started with-rsc-role=Master
pcs constraint colocation add openstack-nova-conductor-clone with openstack-nova-scheduler-clone id=colocation-openstack-nova-conductor-clone-openstack-nova-scheduler-clone-INFINITY rsc-role=Started with-rsc-role=Master
pcs constraint colocation add openstack-nova-novncproxy-clone with openstack-nova-consoleauth-clone id=colocation-openstack-nova-novncproxy-clone-openstack-nova-consoleauth-clone-INFINITY rsc-role=Started with-rsc-role=Master
```


### Neutron

```
mysql -u root -p'CHANGEME' -e "CREATE DATABASE neutron;"
mysql -u root -p'CHANGEME' -e "GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'CHANGEME'; FLUSH PRIVILEGES"
mysql -u root -p'CHANGEME' -e "GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'CHANGEME'; FLUSH PRIVILEGES"


openstack user create --domain default --password CHANGEME neutron
openstack role add --project service --user neutron admin
openstack service create --name neutron --description "OpenStack Networking" network


openstack endpoint create --region RegionOne network public http://api.localdomain:9696
openstack endpoint create --region RegionOne network internal http://api.localdomain:9696
openstack endpoint create --region RegionOne network admin http://api.localdomain:9696

yum install -y openstack-neutron openstack-neutron-ml2 \
  openstack-neutron-linuxbridge ebtables


cat << EOF > /etc/neutron/neutron.conf
[DEFAULT]
transport_url = rabbit://rabbit-user:CHANGEME@controllernode00.localdomain:5672,rabbit-user:CHANGEME@controllernode01.localdomain:5672,rabbit-user:CHANGEME@controllernode02.localdomain:5672/
default_availability_zones = nova
bind_host = 192.168.24.2
auth_strategy = keystone
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
l3_ha = True
l3_ha_net_cidr = 168.254.192.0/18
max_l3_agents_per_router = 3
min_l3_agents_per_router = 2
dhcp_agents_per_network = 3
[agent]
root_helper = sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf
[database]
connection = mysql+pymysql://neutron:CHANGEME@db.localdomain/neutron
[keystone_authtoken]
auth_uri = http://api.localdomain:5000
auth_url = http://api.localdomain:35357
memcached_servers = controllernode00.localdomain:11211,controllernode01.localdomain:11211,controllernode02.localdomain:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = CHANGEME
region_name = RegionOne
[nova]
auth_url = http://api.localdomain:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = CHANGEME
region_name = RegionOne
EOF


sed -i 's/192.168.24.2/192.168.24.3/' /etc/neutron/neutron.conf
sed -i 's/192.168.24.2/192.168.24.4/' /etc/neutron/neutron.conf


cat << EOF > /etc/neutron/plugins/ml2/ml2_conf.ini
[DEFAULT]
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types =
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security
[ml2_type_flat]
flat_networks = provider_flat
[ml2_type_vlan]
network_vlan_ranges = provider_flat
[linux_bridge]
physical_interface_mappings = provider_flat:eth1
[ml2_type_vxlan]
vni_ranges = 1:1000
EOF


cat << EOF > /etc/neutron/plugins/ml2/linuxbridge_agent.ini
[DEFAULT]
[linux_bridge]
physical_interface_mappings = provider_flat:eth1
[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
[vxlan]
enable_vxlan = true
local_ip = 192.168.24.2
l2_population = true
EOF


sed -i 's/192.168.24.2/192.168.24.3/' /etc/neutron/plugins/ml2/linuxbridge_agent.ini
sed -i 's/192.168.24.2/192.168.24.4/' /etc/neutron/plugins/ml2/linuxbridge_agent.ini


echo br_netfilter > /etc/modules-load.d/br_netfilter.conf

modprobe br_netfilter


cat << EOF >> /etc/sysctl.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
EOF


sysctl -p


cat << EOF > /etc/neutron/l3_agent.ini
[DEFAULT]
interface_driver = linuxbridge
vrrp_confs = $state_path/vrrp
EOF


cat << EOF > /etc/neutron/dhcp_agent.ini
[DEFAULT]
interface_driver = linuxbridge
enable_isolated_metadata = true
force_metadata = True
EOF


cat << EOF > /etc/neutron/metadata_agent.ini
[DEFAULT]
nova_metadata_host = api.localdomain
metadata_proxy_shared_secret = METADATA_SECRET
EOF


ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini


su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron


systemctl disable neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service neutron-l3-agent.service

systemctl stop neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service neutron-l3-agent.service
```

#### neutron resource create
```
pcs resource create neutron-server systemd:neutron-server meta interleave=true op monitor start-delay=10s --clone
pcs resource create neutron-linuxbridge-agent systemd:neutron-linuxbridge-agent meta interleave=true op monitor start-delay=10s --clone
pcs resource create neutron-dhcp-agent systemd:neutron-dhcp-agent meta interleave=true op monitor start-delay=10s --clone
pcs resource create neutron-metadata-agent systemd:neutron-metadata-agent meta interleave=true op monitor start-delay=10s --clone
pcs resource create neutron-l3-agent systemd:neutron-l3-agent meta interleave=true op monitor start-delay=10s --clone
pcs resource create neutron-netns-cleanup systemd:neutron-netns-cleanup meta interleave=true op monitor start-delay=10s --clone
pcs resource create neutron-linuxbridge-cleanup systemd:neutron-linuxbridge-cleanup meta interleave=true op monitor start-delay=10s --clone


pcs constraint order httpd-clone then neutron-server-clone id=order-httpd-clone-neutron-server-clone-mandatory
pcs constraint order neutron-server-clone then neutron-linuxbridge-agent-clone id=order-neutron-server-clone-neutron-linuxbridge-agent-clone-mandatory
pcs constraint order neutron-linuxbridge-agent-clone then neutron-dhcp-agent-clone id=order-neutron-linuxbridge-agent-clone-neutron-dhcp-agent-clone-mandatory
pcs constraint order neutron-netns-cleanup-clone then neutron-linuxbridge-agent-clone id=order-neutron-netns-cleanup-clone-neutron-linuxbrige-agent-clone-mandatory
pcs constraint order neutron-l3-agent-clone then neutron-metadata-agent-clone id=order-neutron-l3-agent-clone-neutron-metadata-agent-clone-mandatory
pcs constraint order neutron-dhcp-agent-clone then neutron-l3-agent-clone id=order-neutron-dhcp-agent-clone-neutron-l3-agent-clone-mandatory
pcs constraint order neutron-linuxbridge-cleanup-clone then neutron-netns-cleanup-clone id=order-neutron-linuxbridge-cleanup-clone-neutron-netns-cleanup-clone-mandatory


pcs constraint colocation add neutron-dhcp-agent-clone with neutron-linuxbridge-agent-clone id=colocation-neutron-dhcp-agent-clone-neutron-linuxbridge-agent-clone-INFINITY rsc-role=Started with-rsc-role=Master
pcs constraint colocation add neutron-l3-agent-clone with neutron-dhcp-agent-clone id=colocation-neutron-l3-agent-clone-neutron-dhcp-agent-clone-INFINITY rsc-role=Started with-rsc-role=Master
pcs constraint colocation add neutron-linuxbridge-agent-clone with neutron-netns-cleanup-clone id=colocation-neutron-linuxbridge-agent-clone-neutron-netns-cleanup-clone-INFINITY rsc-role=Started with-rsc-role=Master
pcs constraint colocation add neutron-netns-cleanup-clone with neutron-linuxbridge-cleanup-clone id=colocation-neutron-netns-cleanup-clone-neutron-linuxbridge-cleanup-clone-INFINITY rsc-role=Started with-rsc-role=Master
pcs constraint colocation add neutron-metadata-agent-clone with neutron-l3-agent-clone id=colocation-neutron-metadata-agent-clone-neutron-l3-agent-clone-INFINITY rsc-role=Started with-rsc-role=Master
```

### Cinder

```
mysql -u root -p'CHANGEME' -e "CREATE DATABASE cinder;"
mysql -u root -p'CHANGEME' -e "GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'CHANGEME'; FLUSH PRIVILEGES"
mysql -u root -p'CHANGEME' -e "GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'CHANGEME'; FLUSH PRIVILEGES"


openstack user create --domain default --password CHANGEME cinder
openstack role add --project service --user cinder admin
openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3


openstack endpoint create --region RegionOne volumev2 public http://api.localdomain:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev2 internal http://api.localdomain:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev2 admin http://api.localdomain:8776/v2/%\(project_id\)s

openstack endpoint create --region RegionOne volumev3 public http://api.localdomain:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 internal http://api.localdomain:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 admin http://api.localdomain:8776/v3/%\(project_id\)s


yum install -y openstack-cinder


cat << EOF > /etc/cinder/cinder.conf
[DEFAULT]
rootwrap_config = /etc/cinder/rootwrap.conf
api_paste_confg = /etc/cinder/api-paste.ini
volume_name_template = volume-%s
auth_strategy = keystone
state_path = /var/lib/cinder
lock_path = /var/lock/cinder
volumes_dir = /var/lib/cinder/volumes
enabled_backends = ceph
default_volume_type = ceph
max_over_subscription_ratio = 1000
osapi_volume_listen=192.168.24.2
transport_url = rabbit://rabbit-user:CHANGEME@controllernode00.localdomain:5672,rabbit-user:CHANGEME@controllernode01.localdomain:5672,rabbit-user:CHANGEME@controllernode02.localdomain:5672/
my_ip = 192.168.24.2
[database]
connection = mysql+pymysql://cinder:CHANGEME@db.localdomain/cinder
[keystone_authtoken]
auth_uri = http://api.localdomain:5000
auth_url = http://api.localdomain:35357
memcached_servers = controllernode00.localdomain:11211,controllernode01.localdomain:11211,controllernode02.localdomain:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = CHANGEME
[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
[oslo_messaging_notifications]
driver = messagingv2
[ceph]
volume_driver = "cinder.volume.drivers.rbd.RBDDriver"
volume_backend_name = "ceph"
rbd_cluster_name = "ceph"
rbd_pool = "volumes"
rbd_ceph_conf = "/etc/ceph/ceph.conf"
rbd_flatten_volume_from_snapshot = "false"
rbd_max_clone_depth = "5"
rbd_store_chunk_size = "4"
rados_connect_timeout = "-1"
glance_api_version = "2"
rbd_user = "cinder"
rbd_secret_uuid = "db690dd9-f395-47c2-bcf9-49e6de71b4dc"
EOF


sed -i 's/192.168.24.2/192.168.24.3/g' /etc/cinder/cinder.conf
sed -i 's/192.168.24.2/192.168.24.4/g' /etc/cinder/cinder.conf


su -s /bin/sh -c "cinder-manage db sync" cinder


systemctl disable openstack-cinder-api.service openstack-cinder-scheduler.service openstack-cinder-volume.service
systemctl stop openstack-cinder-api.service openstack-cinder-scheduler.service openstack-cinder-volume.service
```

#### cinder resource create
```
pcs resource create openstack-cinder-api systemd:openstack-cinder-api meta interleave=true op monitor start-delay=60s --clone
pcs resource create openstack-cinder-scheduler systemd:openstack-cinder-scheduler meta interleave=true op monitor start-delay=60s --clone
pcs resource create openstack-cinder-volume systemd:openstack-cinder-volume op monitor start-delay=60s


pcs constraint order httpd-clone then openstack-cinder-api-clone id=order-httpd-clone-openstack-cinder-api-clone-mandatory
pcs constraint order openstack-cinder-scheduler-clone then openstack-cinder-volume id=order-openstack-cinder-scheduler-clone-openstack-cinder-volume-mandatory
pcs constraint order openstack-cinder-api-clone then openstack-cinder-scheduler-clone id=order-openstack-cinder-api-clone-openstack-cinder-scheduler-clone-mandatory


pcs constraint colocation add openstack-cinder-volume with openstack-cinder-scheduler-clone id=colocation-openstack-cinder-volume-openstack-cinder-scheduler-clone-INFINITY rsc-role=Started with-rsc-role=Master
pcs constraint colocation add openstack-cinder-scheduler-clone with openstack-cinder-api-clone id=colocation-openstack-cinder-scheduler-clone-openstack-cinder-api-clone-INFINITY rsc-role=Started with-rsc-role=Master
```

### Heat

```
mysql -u root -p'CHANGEME' -e "CREATE DATABASE heat;"
mysql -u root -p'CHANGEME' -e "GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'%' IDENTIFIED BY 'CHANGEME'; FLUSH PRIVILEGES"
mysql -u root -p'CHANGEME' -e "GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'localhost' IDENTIFIED BY 'CHANGEME'; FLUSH PRIVILEGES"


openstack user create --domain default --password CHANGEME heat
openstack role add --project service --user heat admin
openstack service create --name heat --description "Orchestration" orchestration
openstack service create --name heat-cfn --description "Orchestration" cloudformation


openstack domain create --description "Stack projects and users" heat
openstack user create --domain heat --password CHANGEME heat_domain_admin
openstack role add --domain heat --user heat_domain_admin admin
openstack role add --domain heat --user-domain heat --user heat_domain_admin admin


openstack role create heat_stack_owner
openstack role create heat_stack_user


openstack endpoint create --region RegionOne orchestration public http://api.localdomain:8004/v1/%\(tenant_id\)s
openstack endpoint create --region RegionOne orchestration internal http://api.localdomain:8004/v1/%\(tenant_id\)s
openstack endpoint create --region RegionOne orchestration admin http://api.localdomain:8004/v1/%\(tenant_id\)s

openstack endpoint create --region RegionOne cloudformation public http://api.localdomain:8000/v1
openstack endpoint create --region RegionOne cloudformation internal http://api.localdomain:8000/v1
openstack endpoint create --region RegionOne cloudformation admin http://api.localdomain:8000/v1


yum install -y openstack-heat-api openstack-heat-api-cfn \
  openstack-heat-engine


cat << EOF > /etc/heat/heat.conf
[DEFAULT]
transport_url = rabbit://rabbit-user:CHANGEME@controllernode00.localdomain:5672,rabbit-user:CHANGEME@controllernode01.localdomain:5672,rabbit-user:CHANGEME@controllernode02.localdomain:5672/
heat_metadata_server_url = http://api.localdomain:8000
heat_waitcondition_server_url = http://api.localdomain:8000/v1/waitcondition
stack_domain_admin = heat_domain_admin
stack_domain_admin_password = CHANGEME
stack_user_domain_name = heat
stack_action_timeout = 86400
[clients_keystone]
auth_uri = http://api.localdomain:35357
[database]
connection = mysql+pymysql://heat:CHANGEME@db.localdomain/heat
[ec2authtoken]
auth_uri = http://api.localdomain:5000
[keystone_authtoken]
auth_uri = http://api.localdomain:5000
auth_url = http://api.localdomain:35357
memcached_servers = controllernode00.localdomain:11211,controllernode01.localdomain:11211,controllernode02.localdomain:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = heat
password = CHANGEME
[trustee]
auth_type = password
auth_url = http://api.localdomain:35357
username = heat
password = CHANGEME
user_domain_name = default
[heat_api]
bind_host = 192.168.24.2
[heat_api_cfn]
bind_host = 192.168.24.2
EOF


sed -i 's/192.168.24.2/192.168.24.3/g' /etc/heat/heat.conf
sed -i 's/192.168.24.2/192.168.24.4/g' /etc/heat/heat.conf


su -s /bin/sh -c "heat-manage db_sync" heat


systemctl disable openstack-heat-api.service \
  openstack-heat-api-cfn.service openstack-heat-engine.service
systemctl stop openstack-heat-api.service \
  openstack-heat-api-cfn.service openstack-heat-engine.service
```

#### heat resource create
```
pcs resource create openstack-heat-api systemd:openstack-heat-api meta interleave=true op monitor start-delay=60s --clone
pcs resource create openstack-heat-api-cfn systemd:openstack-heat-api-cfn meta interleave=true op monitor start-delay=60s --clone
pcs resource create openstack-heat-engine systemd:openstack-heat-engine meta interleave=true op monitor start-delay=60s --clone

pcs constraint order httpd-clone then openstack-heat-api-clone id=order-httpd-clone-openstack-heat-api-clone-mandatory
pcs constraint order openstack-heat-api-clone then openstack-heat-api-cfn-clone id=order-openstack-heat-api-clone-openstack-heat-api-cfn-clone-mandatory
pcs constraint order openstack-heat-api-cfn-clone then openstack-heat-engine-clone id=openstack-heat-api-cfn-clone-openstack-heat-engine-clone-mandatory

pcs constraint colocation add openstack-heat-api-cfn-clone with openstack-heat-api-clone id=colocation-openstack-heat-api-cfn-clone-openstack-heat-api-clone-INFINITY rsc-role=Started with-rsc-role=Master
pcs constraint colocation add openstack-heat-api-clone with openstack-heat-engine-clone id=colocation-openstack-heat-api-clone-openstack-heat-engine-clone-INFINITY rsc-role=Started with-rsc-role=Master
```
---

## Post install

### Ceph integration

```
yum install -y python-rbd ceph-common


cat << EOF > /etc/ceph/ceph.conf
[global]
fsid = 91b75b16-94a5-412b-aa34-ac367553ac5c
mon_initial_members = controllernode00,controllernode01,controllernode02
mon_host = 192.168.24.2,192.168.24.3,192.168.24.4
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx


[client]
rbd_cache = true
rbd_cache_writethrough_until_flush = true
admin_socket = /var/run/ceph/$cluster-$type.$id.$pid.$cctid.asok
log_file = /var/log/qemu/qemu-guest-$pid.log
rbd_concurrent_management_ops = 20
EOF


cat << EOF > /etc/ceph/ceph.client.glance.keyring
[client.glance]
        key = AQDFED5cwTBwNxAAjUY2JvYo43ISYC1+kBcBkA==
EOF

cat << EOF > /etc/ceph/ceph.client.nova.keyring
[client.nova]
        key = AQDFED5cwTBwNxAAjUY2JvYo43ISYC1+kBcBkA==
EOF

cat << EOF > /etc/ceph/ceph.client.cinder.keyring
[client.cinder]
        key = AQDFED5cwTBwNxAAjUY2JvYo43ISYC1+kBcBkA==
EOF

* change the key with the right user key

```

### Tunning

* Increase open limit files to RabbitMQ, MariaDB, httpd, haproxy services.
---

## Configure the compute nodes

### Nova Compute

```
yum install -y openstack-nova-compute


cat << EOF > /etc/nova/nova.conf
[DEFAULT]
transport_url = rabbit://rabbit-user:CHANGEME@controllernode00.localdomain:5672,rabbit-user:CHANGEME@controllernode01.localdomain:5672,rabbit-user:CHANGEME@controllernode02.localdomain:5672/
my_ip = 192.168.24.5
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
state_path=/var/lib/nova
enabled_apis=osapi_compute,metadata
firewall_driver = nova.virt.firewall.NoopFirewallDriver
[api]
auth_strategy = keystone
[cells]
enable=false
[glance]
api_servers = http://api.localdomain:9292
[keystone_authtoken]
auth_uri = http://api.localdomain:5000
auth_url = http://api.localdomain:35357
memcached_servers = controllernode00.localdomain:11211,controllernode01.localdomain:11211,controllernode02.localdomain:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = CHANGEME
[neutron]
url = http://api.localdomain:9696
auth_url = http://api.localdomain:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = CHANGEME
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
[placement]
os_region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://api.localdomain:35357/v3
username = placement
password = CHANGEME
[vnc]
enabled = True
vncserver_listen = \$my_ip
vncserver_proxyclient_address = \$my_ip
novncproxy_base_url = http://api.localdomain:6080/vnc_auto.html
[wsgi]
api_paste_config=/etc/nova/api-paste.ini
[libvirt]
virt_type = kvm
images_type = rbd
images_rbd_pool = nova
images_rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = nova
rbd_secret_uuid = db690dd9-f395-47c2-bcf9-49e6de71b4dc
inject_password = false
inject_key = false
inject_partition = -2
live_migration_flag = "VIR_MIGRATE_UNDEFINE_SOURCE,VIR_MIGRATE_PEER2PEER,VIR_MIGRATE_LIVE,VIR_MIGRATE_PERSIST_DEST,VIR_MIGRATE_TUNNELLED"
disk_cachemodes = "none"
[scheduler]
discover_hosts_in_cells_interval = 300
EOF


* change virt_type to qemu if it's a virtual lab
* change my_ip option with node ip address


systemctl enable libvirtd.service openstack-nova-compute.service
systemctl start libvirtd.service openstack-nova-compute.service
```

### Neutron Agent

```
yum install -y openstack-neutron-linuxbridge ebtables ipset


cat << EOF > /etc/neutron/neutron.conf
[DEFAULT]
transport_url = rabbit://rabbit-user:CHANGEME@controllernode00.localdomain:5672,rabbit-user:CHANGEME@controllernode01.localdomain:5672,rabbit-user:CHANGEME@controllernode02.localdomain:5672/
auth_strategy = keystone
core_plugin = ml2
[agent]
root_helper = sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf
[keystone_authtoken]
auth_uri = http://api.localdomain:5000
auth_url = http://api.localdomain:35357
memcached_servers = controllernode00.localdomain:11211,controllernode01.localdomain:11211,controllernode02.localdomain:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = CHANGEME
EOF


cat << EOF > /etc/neutron/plugins/ml2/linuxbridge_agent.ini
[DEFAULT]
[linux_bridge]
physical_interface_mappings = provider_flat:eth1
[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
[vxlan]
enable_vxlan = true
local_ip = 192.168.24.5
l2_population = true
EOF


** change local_ip option with node ip address


systemctl enable neutron-linuxbridge-agent.service
systemctl start neutron-linuxbridge-agent.service
```

### Configure libvirtd to talk to ceph

```
yum install -y python-rbd ceph-common

uuidgen

cat << EOF > /etc/ceph/secret.xml
<secret ephemeral='no' private='no'>
  <uuid>db690dd9-f395-47c2-bcf9-49e6de71b4dc</uuid>
  <usage type='ceph'>
    <name>client.cinder secret</name>
  </usage>
</secret>
EOF


cat << EOF > /etc/ceph/ceph.conf
[global]
fsid = 91b75b16-94a5-412b-aa34-ac367553ac5c
mon_initial_members = controllernode00,controllernode01,controllernode02
mon_host = 192.168.24.2,192.168.24.3,192.168.24.4
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx


[client]
rbd_cache = true
rbd_cache_writethrough_until_flush = true
admin_socket = /var/run/ceph/$cluster-$type.$id.$pid.$cctid.asok
log_file = /var/log/qemu/qemu-guest-$pid.log
rbd_concurrent_management_ops = 20
EOF


cat << EOF > /etc/ceph/ceph.client.cinder.keyring
[client.cinder]
        key = AQDFED5cwTBwNxAAjUY2JvYo43ISYC1+kBcBkA==
EOF


virsh secret-define --file /etc/ceph/secret.xml


virsh secret-set-value --secret db690dd9-f395-47c2-bcf9-49e6de71b4dc --base64 $(awk '/key/ {print $3}' /etc/ceph/ceph.client.cinder.keyring)
```
---
#### That's all =)
