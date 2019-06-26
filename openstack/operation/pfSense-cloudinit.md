---
layout: default
title: pfSense Cloudinit
parent: Operation
gran_parent: OpenStack
nav_order: 2
---

# pfSense Cloudinit to OpenStack Cloud

---

## pfSense installation

Install pfSense 2.4.3 or later with default settings on two partition (/boot and /). Be aware that there isn't any disk partition after the root partition, otherwise, the disk resize won't work.

Configure a single interface as WAN interface.

## pfSense WebGUI customization

1. System / Advanced / Admin Access tab:
* Check: Disable HTTP_REFERER enforcement check
* Check: Enable Secure Shell
* Check: Disable password login for Secure Shell (RSA/DSA key only)

2. System / Advanced / Networking tab:
* Check: Disable hardware checksum offload

## pfSense Console

### Comment the stty lines:
```
vi /etc/phpshellsessions/changepassword
...
// If the user does exist, prompt for password
while (empty($password)) {
        echo gettext("New Password") . ": ";
        //exec('/bin/stty -echo');
        $password = trim(fgets($fp));
        //exec('/bin/stty echo');
        echo "\n";
}

// Confirm password
while (empty($confpassword)) {
        echo gettext("Confirm New Password") . ": ";
        //exec('/bin/stty -echo');
        $confpassword = trim(fgets($fp));
        //exec('/bin/stty echo');
        echo "\n";
}
...
```

### Install cloudinit
```
fetch https://raw.githubusercontent.com/rrbarreto/pfsense-cloudinit/master/installer.sh
sh installer.sh
poweroff
```

### OpenStack image create
```
$ openstack image create pfSense --file pfsense.qcow2 --disk-format qcow2
```
