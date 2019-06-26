---
layout: default
title: pfSense Instance
parent: Operation
gran_parent: OpenStack
nav_order: 3
---

# pfSense as an instance on OpenStack Cloud using Heat.

---

## Stack create

```
$ openstack stack create --parameter key_name=keyname \
--parameter flavor=m1.medium \
--parameter private_net=private \
--parameter public_net=public-fip \
--parameter server_name=pfsense-1 \
--parameter image=pfSense \
--template https://raw.githubusercontent.com/rrbarreto/openstack/master/heat/pfsense-instance.yaml \
pfSense
```

## Retrieve Admin Password

1. From Horizon:
* Select the "Retrieve Password" option from instance menu
* Copy/Paste your Private Key
* Decrypt Password button

2. From command line:
```
$ nova get-password pfsense-1 keyname.pem
```
