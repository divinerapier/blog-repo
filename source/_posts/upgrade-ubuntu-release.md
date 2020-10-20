---
title: 升级 Ubuntu 发行版
date: 2020-10-20 16:05:06
tags:
  - linux
  - ubuntu
---

**Install all available updates for your release before upgrading.**

``` bash
sudo apt update && sudo apt upgrade -y
```

**Reboot system.**

``` bash
sudo reboot
```

**Install the Ubuntu update tool.**

``` bash
sudo apt install -y update-manager-core
```

**Start the upgrade procdure.**

``` bash
sudo do-release-upgrade
```

不建议升级过程通过 **ssh** 连接运行，可以运行在 **tmux** 会话中。

``` bash
$ sudo do-release-upgrade
Checking for a new Ubuntu release
Get:1 Upgrade tool signature [1,554 B]
Get:2 Upgrade tool [1,336 kB]
Fetched 1,338 kB in 0s (0 B/s)
authenticate 'focal.tar.gz' against 'focal.tar.gz.gpg'
extracting 'focal.tar.gz'

Reading cache

Checking package manager

Continue running under SSH?

This session appears to be running under ssh. It is not recommended
to perform a upgrade over ssh currently because in case of failure it
is harder to recover.

If you continue, an additional ssh daemon will be started at port
'1022'.
Do you want to continue?

Continue [yN]
```

**Reboot the box.**

``` bash
sudo reboot
```
