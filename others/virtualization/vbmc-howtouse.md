---
layout: default
title: Virtual BMC
nav_order: 2
parent: vbmc
gran_parent: Others
permalink: /others/virtualization/vbmc-howtouse
---


#  Instalation
---

```
setenforce 0
sed s/SELINUX=.*/SELINUX=disabled/ /etc/selinux/config -i
```
