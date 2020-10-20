---
title: Ubuntu 20.04 配置静态网络
date: 2020-10-19 18:55:44
tags:
  - ubuntu
  - networks
---

**Ubuntu 20.04** 的网络配置文件位于 **/etc/netplan/00-installer-config.yaml**:

``` yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens33:
      dhcp4: false
      addresses: [192.168.50.30/24]
      gateway4: 192.168.50.1
      nameservers:
        addresses: [192.168.50.1, 8.8.8.8]
```

更新配置:

``` bash
root@ubuntu-00:~# netplan --debug apply
** (generate:3768): DEBUG: 11:49:11.734: Processing input file /etc/netplan/00-installer-config.yaml..
** (generate:3768): DEBUG: 11:49:11.734: starting new processing pass
** (generate:3768): DEBUG: 11:49:11.735: We have some netdefs, pass them through a final round of validation
** (generate:3768): DEBUG: 11:49:11.735: ens33: setting default backend to 1
** (generate:3768): DEBUG: 11:49:11.735: Configuration is valid
** (generate:3768): DEBUG: 11:49:11.736: Generating output files..
** (generate:3768): DEBUG: 11:49:11.736: NetworkManager: definition ens33 is not for us (backend 1)
(generate:3768): GLib-DEBUG: 11:49:11.736: posix_spawn avoided (fd close requested)
DEBUG:netplan generated networkd configuration changed, restarting networkd
```
