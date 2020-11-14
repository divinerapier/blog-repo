---
title: 为 Linux 增加新磁盘
date: 2020-11-01 20:41:21
tags:
  - linux
  - hard drive
---

## 查看设备文件

将磁盘插入计算机后，在终端中查看:

``` bash
$ lsblk -d -o name,serial

NAME    SERIAL
nvme0n1 200000000000
```

或者

``` bash
$ sudo fdisk -l

Disk /dev/nvme0n1: 931.53 GiB, 1000204886016 bytes, 1953525168 sectors
Disk model: WDS100T3X0C-00SJG0
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

可以确认设备文件为 `/dev/nvme0n1`。

## 创建分区

创建分区表:

``` bash
$ sudo parted /dev/nvme0n1 mklabel gpt

Information: You may need to update /etc/fstab.
```

创建主分区，并确认对齐:

``` bash
$ sudo parted -s -m /dev/nvme0n1 mkpart primary ext4 1 100%

$ sudo parted /dev/nvme0n1 align-check opt 1
1 aligned
```

查看分区详细信息:

``` bash
$ sudo parted /dev/nvme0n1 print
Model: WDS100T3X0C-00SJG0 (nvme)
Disk /dev/nvme0n1: 1000GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name     Flags
 1      1049kB  1000GB  1000GB               primary
```

## 使用磁盘

创建文件系统:

``` bash
$ sudo mkfs.ext4 /dev/nvme0n1p1

mke2fs 1.45.5 (07-Jan-2020)
Discarding device blocks: done
Creating filesystem with 244190208 4k blocks and 61054976 inodes
Filesystem UUID: b5424944-2d8c-4c5f-8bb4-0e538db5592b
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968,
        102400000, 214990848

Allocating group tables: done
Writing inode tables: done
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done
```

挂载磁盘:

``` bash
$ sudo mkdir -p /nvme

$ sudo mount /dev/nvme0n1p1 /nvme

$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p1  916G   77M  870G   1% /nvme
```

## 测试磁盘

``` bash
$ cd /nvme

# ioengine: 可以指定为 psync / libaio
# numjobs: 测试线程数，线程之间的测试相互独立，成倍占用 size 指定的大小
# rw: 读写方式
#     read: 顺序读
#     write: 顺序写
#     randread: 随机读
#     randwrite: 随机写
# bs: 每次读写块大小
$ sudo fio -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=4k -size=100G -numjobs=4 -group_reporting -name=file
```
