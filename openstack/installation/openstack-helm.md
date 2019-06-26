---
layout: default
title: OpenStack Helm
parent: Installation
gran_parent: OpenStack
nav_order: 3
---

#  Deploy OpenStack with Helm on CentOS using external Ceph

---

## Topoly

* In this topology will be used 5 virtual machines to form a kubernetes cluster being 1 to master and 4 to nodes;
* Helm will be installed on master;
* Nova Compute, Glance, Cinder and Gnocchi on external Ceph;
* Ceph cluster. We won't cover ceph deployment here. I suppose that you use a ceph cluster lab environment to perform those steps.

---

## Prepare all nodes

#### Configure repositories
```
cat << 'EOF' > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

cat << 'EOF' > /etc/yum.repos.d/ceph.repo
[ceph]
name=Ceph packages for $basearch
baseurl=https://download.ceph.com/rpm-luminous/el7/$basearch
enabled=1
priority=2
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc
EOF
```

#### Install packages
```
yum update -y
yum install epel-release -y
yum install vim bash-completion docker kubeadm kubelet kubectl ceph-common -y

systemctl enable kubelet docker
systemctl start docker
```

#### Configuration
```
kubectl completion bash > /etc/bash_completion.d/kubectl.bash
chmod +x /etc/bash_completion.d/kubectl.bash

cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system

cat << EOF > /etc/modules-load.d/rbd.conf
# Load rbd kernel module
rbd
EOF

modprobe rbd

sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
setenforce 0
```

## Deploy Kubernetes cluster

### On master node

#### K8s control plane setup
```
kubeadm config images pull

kubeadm init --apiserver-advertise-address=<<MASTERNODEIP>> --pod-network-cidr=10.244.0.0/16
```
After that the command kubeadm init has fineshed, you will see the output as below:
```
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config

      You should now deploy a pod network to the cluster.
      Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
        https://kubernetes.io/docs/concepts/cluster-administration/addons/

        Then you can join any number of worker nodes by running the following on each as root:

        kubeadm join 192.168.122.10:6443 --token gzco2t.n6g1swh2s5x6rp4g --discovery-token-ca-cert-hash sha256:97b3c8854af5f2889d453143e4d641a592fb55400d8462251ac6c13d1e15b4e3
```
**Save all kubeadm join command to be used to join the k8s nodes in the k8s cluster.**

#### k8s network setup
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### On worker nodes

#### Join nodes to the k8s cluster
```
kubeadm join <<MASTERNODEIP>>:6443 --token gzco2t.n6g1swh2s5x6rp4g --discovery-token-ca-cert-hash sha256:97b3c8854af5f2889d453143e4d641a592fb55400d8462251ac6c13d1e15b4e3
```

## Install Helm
```
curl -o helm-v2.13.1-linux-amd64.tar.gz https://storage.googleapis.com/kubernetes-helm/helm-v2.13.1-linux-amd64.tar.gz

tar xvf helm-v2.13.1-linux-amd64.tar.gz

cp linux-amd64/helm /usr/local/bin/

chmod +x /usr/local/bin/helm

helm completion bash > /etc/bash_completion.d/helm.bash

helm init

kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'

cat << EOF > /etc/systemd/system/helm-serve.service
[Unit]
Description=Helm Server
After=network.target

[Service]
User=root
Restart=always
ExecStart=/usr/local/bin/helm serve

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable helm-serve.service
systemctl start helm-serve.service

helm repo remove local
helm repo remove stable
helm repo add local  http://localhost:8879/charts

curl http://localhost:8879/charts

helm repo update
helm ls
helm search 
```

## Deploy OpenStack

#### Label all nodes
```
kubectl label nodes --all openstack-control-plane=enabled
kubectl label nodes --all openstack-compute-node=enabled
kubectl label nodes --all openvswitch=enabled
kubectl label nodes --all linuxbridge=enabled
kubectl label nodes --all ceph-mon=enabled
kubectl label nodes --all ceph-osd=enabled
kubectl label nodes --all ceph-mds=enabled
kubectl label nodes --all ceph-rgw=enabled
kubectl label nodes --all ceph-mgr=enabled
```

#### Ceph configuration
```
ceph osd pool create kube 128 128
ceph osd pool create images 128 128
ceph osd pool create volumes 128 128
ceph osd pool create nova 128 128

ceph auth add client.kube mon 'allow r' osd 'allow rwx pool=kube'
ceph auth add client.glance mon 'allow r' osd 'allow rwx pool=images'
ceph auth add client.cinder mon 'allow r' osd 'allow rwx pool=images, allow rwx pool=volumes, allow rwx pool=nova'

ceph auth get-key client.kube |base64
ceph auth get-key client.admin |base64

* We'll use this to create the secrets, save it.

```

#### Secrets and storageclass
```
cat << EOF > rbd-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: general
  namespace: kube-system
provisioner: ceph.com/rbd
parameters:
  monitors: <<CEPHMONITOR1>>:6789,<<CEPHMONITOR2>>:6789,<<CEPHMONITOR3>>:6789
  adminId: admin
  adminSecretName: ceph-admin-secret
  adminSecretNamespace: openstack
  pool: kube
  userId: kube
  userSecretName: ceph-kube-secret
  userSecretNamespace: openstack
  imageFormat: "2"
  imageFeatures: layering
EOF

cat << EOF > ceph-admin-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: "ceph-admin-secret"
  namespace: openstack
type: kubernetes.io/rbd
data:
  key: "<<ADMINKEYRINGBASE64>>"
EOF

cat << EOF > ceph-kube-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: "ceph-kube-secret"
  namespace: openstack
type: kubernetes.io/rbd
data:
  key: "<<KUBEKEYRINGBASE64>>"
EOF

kubectl create namespace openstack
kubectl apply --namespace openstack -f ceph-kube-secret.yaml
kubectl apply --namespace openstack -f ceph-admin-secret.yaml
kubectl create -f rbd-storageclass.yaml
```

#### Openstack helm repositories
```
git clone https://opendev.org/openstack/openstack-helm-infra.git
git clone https://opendev.org/openstack/openstack-helm.git
```

#### Deployment customization
```
mkdir ./openstack-helm/tools/deployment/developer/os-lab
cd ./openstack-helm/tools/deployment/developer/os-lab
```

Create the custom scripts as follow:

##### 020-setup-client.sh
```
#!/bin/bash
set -xe

sudo -H -E pip install "cmd2<=0.8.7"
sudo -H -E pip install python-openstackclient python-heatclient --ignore-installed

sudo -H mkdir -p /etc/openstack
sudo -H chown -R $(id -un): /etc/openstack
tee /etc/openstack/clouds.yaml << EOF
clouds:
  openstack_helm:
    region_name: RegionOne
    identity_api_version: 3
    auth:
      username: 'admin'
      password: 'password'
      project_name: 'admin'
      project_domain_name: 'default'
      user_domain_name: 'default'
      auth_url: 'http://keystone.openstack.svc.cluster.local/v3'
EOF

#NOTE: Build charts
make all
```

##### 030-ingress.sh
```
#!/bin/bash
set -xe

#NOTE: Lint and package chart
: ${OSH_INFRA_PATH:="../openstack-helm-infra"}
make -C ${OSH_INFRA_PATH} ingress

#NOTE: Deploy command
: ${OSH_INFRA_PATH:="../openstack-helm-infra"}
: ${OSH_EXTRA_HELM_ARGS:=""}
tee /tmp/ingress-kube-system.yaml << EOF
deployment:
  mode: cluster
  type: DaemonSet
network:
  host_namespace: true
EOF
helm upgrade --install ingress-kube-system ${OSH_INFRA_PATH}/ingress \
  --namespace=kube-system \
  --values=/tmp/ingress-kube-system.yaml \
  ${OSH_EXTRA_HELM_ARGS} \
  ${OSH_EXTRA_HELM_ARGS_INGRESS_KUBE_SYSTEM}

#NOTE: Wait for deploy
./tools/deployment/common/wait-for-pods.sh kube-system

#NOTE: Display info
helm status ingress-kube-system

#NOTE: Deploy namespace ingress
helm upgrade --install ingress-openstack ${OSH_INFRA_PATH}/ingress \
  --namespace=openstack \
  ${OSH_EXTRA_HELM_ARGS} \
  ${OSH_EXTRA_HELM_ARGS_INGRESS_OPENSTACK}  

#NOTE: Wait for deploy
./tools/deployment/common/wait-for-pods.sh openstack

#NOTE: Display info
helm status ingress-openstack

helm upgrade --install ingress-ceph ${OSH_INFRA_PATH}/ingress \
  --namespace=ceph \
  ${OSH_EXTRA_HELM_ARGS} \
  ${OSH_EXTRA_HELM_ARGS_INGRESS_CEPH}

#NOTE: Wait for deploy
./tools/deployment/common/wait-for-pods.sh ceph

#NOTE: Display info
helm status ingress-ceph
```

##### 040-ceph-provisioners-deploy-with-existing-ceph.sh
```
#!/bin/bash
set -xe

#NOTE: Lint and package chart
: ${OSH_INFRA_PATH:="../openstack-helm-infra"}
make -C ${OSH_INFRA_PATH} ceph-provisioners 

#NOTE: Deploy command
: ${OSH_EXTRA_HELM_ARGS:=""}

#Change YOUR_CEPH_IP to your ceph-mon ip address
tee /tmp/ceph-openstack-config.yaml <<EOF   
deployment:
  ceph: true
  client_secrets: true
  rbd_provisioner: true
  cephfs_provisioner: true
bootstrap:
  enabled: false
dependencies:
  static:
    cephfs_provisioner:
      jobs: null
      services: null
    rbd_provisioner:
      jobs: null
      services: null
storageclass:
  rbd:
    provision_storage_class: false
  cephfs:
    provision_storage_class: false
conf:
  ceph:
    global:
      mon_host: <<CEPHMONITOR1>>
      mon_addr: <<CEPHMONITOR1>>:6789
    osd:
      cluster_network: 10.244.0.0/16
      public_network: 10.244.0.0/16
manifests:
  configmap_bin: true
  configmap_bin_common: true
  configmap_etc: true
  deployment_rbd_provisioner: true
  deployment_cephfs_provisioner: true
  job_bootstrap: false
  job_cephfs_client_key: false
  job_image_repo_sync: true
  job_namespace_client_key_cleaner: false   
  job_namespace_client_key: false
  storageclass_cephfs: false
  storageclass_rbd: false
EOF
: ${OSH_EXTRA_HELM_ARGS:=""}

helm upgrade --install ceph-openstack-config ${OSH_INFRA_PATH}/ceph-provisioners \
  --namespace=openstack \
  --values=/tmp/ceph-openstack-config.yaml \
  ${OSH_EXTRA_HELM_ARGS}
```

##### 050-mariadb.sh
```
#!/bin/bash
set -xe

#NOTE: Lint and package chart
: ${OSH_INFRA_PATH:="../openstack-helm-infra"}
make -C ${OSH_INFRA_PATH} mariadb

#NOTE: Deploy command
: ${OSH_EXTRA_HELM_ARGS:=""}
helm upgrade --install mariadb ${OSH_INFRA_PATH}/mariadb \
    --namespace=openstack \
    --set pod.replicas.server=1 \
    ${OSH_EXTRA_HELM_ARGS} \
    ${OSH_EXTRA_HELM_ARGS_MARIADB}

#NOTE: Wait for deploy
./tools/deployment/common/wait-for-pods.sh openstack

#NOTE: Validate Deployment info
helm status mariadb
```

##### 060-rabbitmq.sh
```
#!/bin/bash
set -xe

#NOTE: Lint and package chart
: ${OSH_INFRA_PATH:="../openstack-helm-infra"}
make -C ${OSH_INFRA_PATH} rabbitmq

#NOTE: Deploy command
: ${OSH_EXTRA_HELM_ARGS:=""}
helm upgrade --install rabbitmq ${OSH_INFRA_PATH}/rabbitmq \
    --namespace=openstack \
    --set pod.replicas.server=1 \
    ${OSH_EXTRA_HELM_ARGS} \
    ${OSH_EXTRA_HELM_ARGS_RABBITMQ}

#NOTE: Wait for deploy
./tools/deployment/common/wait-for-pods.sh openstack

#NOTE: Validate Deployment info
helm status rabbitmq
```

##### 070-memcached.sh
```
#!/bin/bash
set -xe

#NOTE: Lint and package chart
: ${OSH_INFRA_PATH:="../openstack-helm-infra"}
make -C ${OSH_INFRA_PATH} memcached

tee /tmp/memcached.yaml <<EOF
manifests:
  network_policy: true
network_policy:
  memcached:
    ingress:
      - from:
        - podSelector:
            matchLabels:
              application: keystone
        - podSelector:
            matchLabels:
              application: heat
        - podSelector:
            matchLabels:
              application: glance
        - podSelector:
            matchLabels:
              application: cinder
        - podSelector:
            matchLabels:
              application: congress
        - podSelector:
            matchLabels:
              application: barbican
        - podSelector:
            matchLabels:
              application: ceilometer
        - podSelector:
            matchLabels:
              application: horizon
        - podSelector:
            matchLabels:
              application: ironic
        - podSelector:
            matchLabels:
              application: magnum
        - podSelector:
            matchLabels:
              application: mistral
        - podSelector:
            matchLabels:
              application: nova
        - podSelector:
            matchLabels:
              application: neutron
        - podSelector:
            matchLabels:
              application: senlin
        ports:
        - protocol: TCP
          port: 11211
EOF

#NOTE: Deploy command
: ${OSH_EXTRA_HELM_ARGS:=""}
helm upgrade --install memcached ${OSH_INFRA_PATH}/memcached \
    --namespace=openstack \
    --values=/tmp/memcached.yaml \
    ${OSH_EXTRA_HELM_ARGS} \
    ${OSH_EXTRA_HELM_ARGS_MEMCACHED}

#NOTE: Wait for deploy
./tools/deployment/common/wait-for-pods.sh openstack

#NOTE: Validate Deployment info
helm status memcached
```

##### 080-keystone.sh
```
#!/bin/bash
set -xe

#NOTE: Lint and package chart
make keystone

#NOTE: Deploy command
: ${OSH_EXTRA_HELM_ARGS:=""}
helm upgrade --install keystone ./keystone \
    --namespace=openstack \
    ${OSH_EXTRA_HELM_ARGS} \
    ${OSH_EXTRA_HELM_ARGS_KEYSTONE}

#NOTE: Wait for deploy
./tools/deployment/common/wait-for-pods.sh openstack

#NOTE: Validate Deployment info
helm status keystone
export OS_CLOUD=openstack_helm
sleep 30 #NOTE(portdirect): Wait for ingress controller to update rules and restart Nginx
openstack endpoint list
```

##### 090-heat.sh
```
#!/bin/bash
set -xe

#NOTE: Lint and package chart
make heat

#NOTE: Deploy command
: ${OSH_EXTRA_HELM_ARGS:=""}
helm upgrade --install heat ./heat \
  --namespace=openstack \
  ${OSH_EXTRA_HELM_ARGS} \
  ${OSH_EXTRA_HELM_ARGS_HEAT}

#NOTE: Wait for deploy
./tools/deployment/common/wait-for-pods.sh openstack

#NOTE: Validate Deployment info
export OS_CLOUD=openstack_helm
openstack service list
sleep 30 #NOTE(portdirect): Wait for ingress controller to update rules and restart Nginx
openstack orchestration service list
```

##### 100-horizon.sh
```
#!/bin/bash
set -xe

#NOTE: Lint and package chart
make horizon

#NOTE: Deploy command
: ${OSH_EXTRA_HELM_ARGS:=""}
helm upgrade --install horizon ./horizon \
    --namespace=openstack \
    --set network.node_port.enabled=true \
    --set network.node_port.port=31000 \
    ${OSH_EXTRA_HELM_ARGS} \
    ${OSH_EXTRA_HELM_ARGS_HORIZON}

#NOTE: Wait for deploy
./tools/deployment/common/wait-for-pods.sh openstack

#NOTE: Validate Deployment info
helm status horizon

helm test horizon
```

##### 120-glance.sh
```
#!/bin/bash
set -xe

#NOTE: Lint and package chart
make glance

#NOTE: Deploy command
: ${OSH_EXTRA_HELM_ARGS:=""}
: ${OSH_OPENSTACK_RELEASE:="newton"}
#NOTE(portdirect), this could be: radosgw, rbd, swift or pvc
: ${GLANCE_BACKEND:="rbd"}
tee /tmp/glance.yaml <<EOF
storage: ${GLANCE_BACKEND}
EOF
if [ "x${OSH_OPENSTACK_RELEASE}" == "xnewton" ]; then
# NOTE(portdirect): glance APIv1 is required for heat in Newton
  tee -a /tmp/glance.yaml <<EOF
conf:
  glance:
    DEFAULT:
      enable_v1_api: true
      enable_v2_registry: true
    glance_store:
      rbd_store_user: glance
      rbd_store_pool: images
  ceph:
    enabled: true
ceph_client:
  configmap: ceph-etc
  user_secret_name: ceph-admin-secret
manifests:
  deployment_registry: true
  ingress_registry: true
  pdb_registry: true
  service_ingress_registry: true
  service_registry: true
  job_storage_init: false
EOF
fi
helm upgrade --install glance ./glance \
  --namespace=openstack \
  --values=/tmp/glance.yaml \
  --set manifests.network_policy=true \
  ${OSH_EXTRA_HELM_ARGS} \
  ${OSH_EXTRA_HELM_ARGS_GLANCE}

#NOTE: Wait for deploy
./tools/deployment/common/wait-for-pods.sh openstack

#NOTE: Validate Deployment info
helm status glance
export OS_CLOUD=openstack_helm
openstack service list
sleep 30 #NOTE(portdirect): Wait for ingress controller to update rules and restart Nginx
openstack image list
openstack image show 'Cirros 0.3.5 64-bit'
```

##### 130-cinder.sh
```
#!/bin/bash
set -xe

#NOTE: Lint and package chart
make cinder

#NOTE: Deploy command
: ${OSH_EXTRA_HELM_ARGS:=""}
tee /tmp/cinder.yaml <<EOF
conf:
  ceph:
    enable: true
  cinder:
    DEFAULT:
      debug: true
      backup_driver: cinder.backup.drivers.ceph
      backup_ceph_user: cinder
      backup_ceph_pool: volumes
  backends:
    rbd1:
      volume_driver: cinder.volume.drivers.rbd.RBDDriver
      volume_backend_name: rbd1
      rbd_ceph_conf: "/etc/ceph/ceph.conf"
      rbd_user: cinder
      rbd_pool: volumes
      rbd_secret_uuid: 33dfcb04-2279-4010-8f0d-dd41f7183c2c
ceph_client:
  configmap: ceph-etc
  user_secret_name: ceph-admin-secret
manifests:
  deployment_backup: false
  job_backup_storage_init: false
  pvc_backup: false
  job_storage_init: false
EOF
helm upgrade --install cinder ./cinder \
  --namespace=openstack \
  --values=/tmp/cinder.yaml \
    --set manifests.network_policy=true \   
  ${OSH_EXTRA_HELM_ARGS} \
  ${OSH_EXTRA_HELM_ARGS_CINDER}

#NOTE: Wait for deploy
./tools/deployment/common/wait-for-pods.sh openstack

#NOTE: Validate Deployment info
export OS_CLOUD=openstack_helm
openstack service list
sleep 30 #NOTE(portdirect): Wait for ingress controller to update rules and restart Nginx
openstack volume type list
```

##### 140-openvswitch.sh
```
#!/bin/bash
set -xe

#NOTE: Lint and package chart
: ${OSH_INFRA_PATH:="../openstack-helm-infra"}
make -C ${OSH_INFRA_PATH} openvswitch

#NOTE: Deploy command
: ${OSH_EXTRA_HELM_ARGS:=""}
helm upgrade --install openvswitch ${OSH_INFRA_PATH}/openvswitch \
  --namespace=openstack \
  ${OSH_EXTRA_HELM_ARGS} \
  ${OSH_EXTRA_HELM_ARGS_OPENVSWITCH}

#NOTE: Wait for deploy
./tools/deployment/common/wait-for-pods.sh openstack

#NOTE: Validate Deployment info
helm status openvswitch
```

##### 150-libvirt.sh
```
#!/bin/bash
set -xe

#NOTE: Lint and package chart
: ${OSH_INFRA_PATH:="../openstack-helm-infra"}
make -C ${OSH_INFRA_PATH} libvirt

tee /tmp/libvirt.yaml <<EOF
manifests:
  network_policy: true
network_policy:
  libvirt:
    ingress:
      - {}
conf:
  ceph:
    enabled: true
    cinder:
       user: cinder
       secret_uuid: 33dfcb04-2279-4010-8f0d-dd41f7183c2c
ceph_client:
  configmap: ceph-etc
  user_secret_name: ceph-admin-secret
EOF

#NOTE: Deploy command
: ${OSH_EXTRA_HELM_ARGS:=""}
helm upgrade --install libvirt ${OSH_INFRA_PATH}/libvirt \
  --namespace=openstack \
  --set manifests.network_policy=true \
  --values=/tmp/libvirt.yaml \
  ${OSH_EXTRA_HELM_ARGS} \
  ${OSH_EXTRA_HELM_ARGS_LIBVIRT}

#NOTE(portdirect): We don't wait for libvirt pods to come up, as they depend
# on the neutron agents being up.

#NOTE: Validate Deployment info
helm status libvirt
```

##### 160-compute-kit.sh
```
#!/bin/bash
set -xe

#NOTE: Lint and package chart
make nova
make neutron

#NOTE: Deploy nova
: ${OSH_EXTRA_HELM_ARGS:=""}

tee /tmp/nova.yaml << EOF
conf:
  ceph:
    enabled: true
    cinder:
      user: cinder
      secret_uuid: 33dfcb04-2279-4010-8f0d-dd41f7183c2c
  nova:
    libvirt:
      images_type: rbd
      images_rbd_pool: nova
      images_rbd_ceph_conf: /etc/ceph/ceph.conf
      rbd_user: cinder
      rbd_secret_uuid: 33dfcb04-2279-4010-8f0d-dd41f7183c2c
ceph_client:
  configmap: ceph-etc
  user_secret_name: ceph-admin-secret
EOF

if [ "x$(systemd-detect-virt)" == "xnone" ]; then
  echo 'OSH is not being deployed in virtualized environment'
  helm upgrade --install nova ./nova \
      --namespace=openstack \
      --values=/tmp/nova.yaml \
      --set manifests.network_policy=true \
      ${OSH_EXTRA_HELM_ARGS} \
      ${OSH_EXTRA_HELM_ARGS_NOVA}
else
  echo 'OSH is being deployed in virtualized environment, using qemu for nova'
  helm upgrade --install nova ./nova \
      --namespace=openstack \
      --values=/tmp/nova.yaml \
      --set conf.nova.libvirt.virt_type=qemu \
      --set conf.nova.libvirt.cpu_mode=none \
      --set manifests.network_policy=true \
      ${OSH_EXTRA_HELM_ARGS} \
      ${OSH_EXTRA_HELM_ARGS_NOVA}
fi

#NOTE: Deploy neutron
tee /tmp/neutron.yaml << EOF
network:
  interface:
    tunnel: docker0
conf:
  neutron:
    DEFAULT:
      l3_ha: False
      max_l3_agents_per_router: 1
      l3_ha_network_type: vxlan
      dhcp_agents_per_network: 1
  plugins:
    ml2_conf:
      ml2_type_flat:
        flat_networks: public
    #NOTE(portdirect): for clarity we include options for all the neutron
    # backends here.
    openvswitch_agent:
      agent:
        tunnel_types: vxlan
      ovs:
        bridge_mappings: public:br-ex
    linuxbridge_agent:
      linux_bridge:
        bridge_mappings: public:br-ex
EOF
helm upgrade --install neutron ./neutron \
    --namespace=openstack \
    --values=/tmp/neutron.yaml \
    --set manifests.network_policy=true \
    ${OSH_EXTRA_HELM_ARGS} \
    ${OSH_EXTRA_HELM_ARGS_NEUTRON}

#NOTE: Wait for deploy
./tools/deployment/common/wait-for-pods.sh openstack

#NOTE: Validate Deployment info
export OS_CLOUD=openstack_helm
openstack service list
sleep 30 #NOTE(portdirect): Wait for ingress controller to update rules and restart Nginx
openstack compute service list
openstack network agent list
```

#### Deploy
```
chmod +x ./openstack-helm/tools/deployment/developer/os-lab/*.sh

cd ./openstack-helm

./tools/deployment/developer/os-lab/020-setup-client.sh
./tools/deployment/developer/os-lab/030-ingress.sh
./tools/deployment/developer/os-lab/040-ceph-provisioners-deploy-with-existing-ceph.sh
./tools/deployment/developer/os-lab/050-mariadb.sh
./tools/deployment/developer/os-lab/060-rabbitmq.sh
./tools/deployment/developer/os-lab/070-memcached.sh
./tools/deployment/developer/os-lab/080-keystone.sh
./tools/deployment/developer/os-lab/090-heat.sh
./tools/deployment/developer/os-lab/100-horizon.sh
./tools/deployment/developer/os-lab/120-glance.sh
./tools/deployment/developer/os-lab/130-cinder.sh
./tools/deployment/developer/os-lab/140-openvswitch.sh
./tools/deployment/developer/os-lab/150-libvirt.sh
./tools/deployment/developer/os-lab/160-compute-kit.sh
```

#### Troubleshooting

##### missing Ceph monitors workaround
```
kind: Service
apiVersion: v1
metadata:
  name: ceph-mon-1
  namespace: ceph
spec:
  type: ExternalName
  externalName: 192.168.0.1.xip.io
---
kind: Service
apiVersion: v1
metadata:
  name: ceph-mon-2
  namespace: ceph
spec:
  type: ExternalName
  externalName: 192.168.0.2.xip.io
---
kind: Service
apiVersion: v1
metadata:
  name: ceph-mon-3
  namespace: ceph
spec:
  type: ExternalName
  externalName: 192.168.0.3.xip.io
```

Then change rbd storage class to use as below:
```
 monitors: ceph-mon-1.ceph.svc.cluster.local.:6789,ceph-mon-2.ceph.svc.cluster.local.:6789,ceph-mon-3.ceph.svc.cluster.local.:6789
```

##### if using job_storage_init and then it got an error when creating the secrets, maybe it could help:
```
--- a/cinder/templates/bin/_storage-init.sh.tpl
+++ b/cinder/templates/bin/_storage-init.sh.tpl
-  kubectl apply --namespace ${NAMESPACE} -f ${SECRET}
+  kubectl apply --namespace ${NAMESPACE} --validate=false -f ${SECRET}

--- a/glance/templates/bin/_storage-init.sh.tpl
+++ b/glance/templates/bin/_storage-init.sh.tpl
-  kubectl apply --namespace "${NAMESPACE}" -f "${SECRET}"
+  kubectl apply --namespace "${NAMESPACE}" --validate=false -f "${SECRET}"
```
---
That's it ;)
