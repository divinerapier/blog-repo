---
title: 挂载 NFS
date: 2020-10-02 16:18:20
tags:
  - nfs
---

查看被分享目录的属性:

``` bash
$ stat /public
  File: /public
  Size: 4096            Blocks: 8          IO Block: 4096   directory
Device: 802h/2050d      Inode: 58982401    Links: 2
Access: (0777/drwxrwxrwx)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2020-10-02 16:25:03.129246127 +0800
Modify: 2020-10-02 16:24:56.129203697 +0800
Change: 2020-10-02 16:24:56.129203697 +0800
 Birth: -
```

``` bash
$ sudo exportfs -avrf
exporting *:/public
```

服务端 **/etc/exports** 配置如下:

``` text
/public *(rw,sync,no_subtree_check)
```

确认被 NFS 导出的本地文件系统:

``` bash
$ sudo exportfs -avrf
exporting *:/public
```

在客户端查看远端配置:

``` bash
$ showmount -e 192.168.50.5
Export list for 192.168.50.5:
/public                               *
```

客户端挂载 NFS 文件系统:

``` bash
sudo mount -o rw,nolock -t nfs 192.168.50.5:/public ./tmp
```

当出现如下报错信息:

``` text
mount.nfs: Operation not permitted
```

请修改 **/etc/exports** 内容为:

``` text
/public *(rw,sync,all_squash,no_subtree_check,insecure)
```

然后重新执行命令挂载。

如果希望指定 `user` 与 `group` 来操作文件，可以通过在 **/etc/exports** 中增加选项: **anonuid=1026,anongid=100**。

## Reference

* [User permissions in NFS mounted directory](https://unix.stackexchange.com/questions/252812/user-permissions-in-nfs-mounted-directory)
* [Mount failed with mount: mount.nfs: access denied by server while mounting error](https://access.redhat.com/solutions/3773891)
