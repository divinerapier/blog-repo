---
title: 容器 - 深入理解镜像
date: 2020-08-20 09:50:18
tags:
  - container
  - docker image
  - mount namespace
  - unionfs
---

`Namespace` 与 `Cgroup` 技术是容器技术的核心点，但 `Docker` 项目的成功关键点却要归功于 `Docker Image` 的发明。在 `Cloud Foundry` 时代，**上云** 的过程需要经过多次 **玄学调参** 才能解决由于本地环境与云主机的差异性所导致的问题。`Docker` 则通过 `Mount Namespace` 与 `UnionFS` 技术，成功的解决了这个问题。

## Mount Namespace

``` bash
$ man mount_namespace
MOUNT_NAMESPACES(7)                 Linux Programmer's Manual                 MOUNT_NAMESPACES(7)

NAME
       mount_namespaces - overview of Linux mount namespaces

DESCRIPTION
       For an overview of namespaces, see namespaces(7).

       Mount namespaces provide isolation of the list of mount points seen by the processes in each
       namespace instance. Thus, the processes in each of the mount namespace instances will see
       distinct single-directory hierarchies.

       The views provided by the /proc/[pid]/mounts, /proc/[pid]/mountinfo, and /proc/[pid]/mountstats
       files (all described in proc(5)) correspond to the mount namespace in which the process with
       the PID [pid] resides. (All of the processes that reside in the same mount namespace will see
       the same view in these files.)

       A new mount namespace is created using either clone(2) or unshare(2) with the CLONE_NEWNS flag.
       When a new mount namespace is created, its mount point list is initialized as follows:

       * If the namespace is created using clone(2), the mount point list of the child's namespace is
         a copy of the mount point list in the parent's namespace.

       * If the namespace is created using unshare(2), the mount point list of the new namespace is a
         copy of the mount point list in the caller's previous mount namespace.

       Subsequent modifications to the mount point list (mount(2) and umount(2)) in either mount
       namespace will not (by default) affect the mount point list seen in the other namespace (but
       see the following discussion of shared subtrees).
```

简单来说，`Mount Namepace` 为进程提供独立的文件系统视图，即可以将进程的文件系统挂载到指定挂载点，从而是进程只能看到 `Mount Namespace` 中的文件系统。

接下来，还是通过代码展示。

### Mount Namespace 开发

下面的代码，使用 `Mount Namespace` 的方式通过 `clone(2)` 系统调用，创建一个新的进程。在该进程中执行 `/bin/bash` 程序。

``` c
#define _GNU_SOURCE

#include <sys/mount.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

#define STACK_SIZE (1024 * 1024)

static char container_stack[STACK_SIZE];

char *const container_args[] = {
        "/bin/bash",
        NULL
};

int container_main(void *arg) {
    printf("Container - inside the container!\n");
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}

int main() {
    printf("Parent - start a container!\n");
    int container_pid = clone(container_main, container_stack + STACK_SIZE, CLONE_NEWNS | SIGCHLD, NULL);
    if (container_pid < 0) {
        perror("failed to create a new process");
        return 1;
    }
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```

编译并运行程序:

``` bash
$ gcc main.c -o mn; sudo ./mn
Parent - start a container!
Container - inside the container!
[root@zephyrus 01-mount-namespace]#
```

如此，就成功的进入到了容器环境内。

*注意*: 是要使用 `root` 权限执行这个程序。

然后，在容器内执行 `df -h` 指令:

``` bash
[root@zephyrus 01-mount-namespace]# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdd        251G   39G  200G  17% /
tools           931G  362G  570G  39% /init
none            2.0G     0  2.0G   0% /dev
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
none            2.0G  8.0K  2.0G   1% /run
none            2.0G     0  2.0G   0% /run/lock
none            2.0G     0  2.0G   0% /run/shm
none            2.0G     0  2.0G   0% /run/user
tmpfs           2.0G     0  2.0G   0% /mnt/wsl
/dev/sdc        251G   11G  228G   5% /mnt/wsl/docker-desktop-data/isocache
none            2.0G   12K  2.0G   1% /mnt/wsl/docker-desktop/shared-sockets/host-services
/dev/sdb        251G  117M  239G   1% /mnt/wsl/docker-desktop/docker-desktop-proxy
/dev/loop0      231M  231M     0 100% /mnt/wsl/docker-desktop/cli-tools
C:\             931G  362G  570G  39% /mnt/c
```

会发现，解决与在宿主机上执行该命令的结果是相同的。这个结果很不好，甚至可以说很危险。因为，不但容器可以看到宿主机上的文件，甚至于还拥有 `root` 权限。接下来，尝试通过在容器中设置挂载点的方式解决这个问题。

### Mount Namespace 指定挂载点

修改 `container_main` 函数:

``` c
int container_main(void *arg) {
    printf("Container - inside the container!\n");
    mount("none", "/tmp", "tmpfs", 0, "");
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}
```

使用 `mount(2)` 系统调用，在容器进程中，增加一个挂载点。

使用 `ls` 指令确认 `/tmp` 目录为空目录，说明挂载成功:

``` bash
$ ls /tmp
```

之后，在确认一下系统的文件系统:

``` bash
# 在原来的文件系统基础之上会多出一个
[root@zephyrus 01-mount-namespace]# df -h
Filesystem      Size  Used Avail Use% Mounted on
none            2.0G     0  2.0G   0% /tmp

[root@zephyrus 01-mount-namespace]# mount -l | grep tmpfs
none on /tmp type tmpfs (rw,relatime)
```

这些都可以说明，已经成功在容器内挂载了一个文件系统。而且，在宿主机上是无法看到这个挂载点的。到目前为止，一切都是按照预期发展的。

### Mount Namespace 挂载根目录

既然可以在容器内部挂载 `/tmp`，那么现在来尝试挂载 `/`。

首先，准备一下必要的环境:

* **bash**: 作为容器的第一个进程，允许在容器执行其他指令
* **ls**: 观察容器内的文件系统是否符合预期
* **lib**: 存放 `bash` 与 `ls` 必须的动态链接库

``` bash
#/bin/bash

T=root
mkdir -p ${T}/{bin,etc,lib,usr}
cp -v /bin/{bash,ls} ${T}/bin
cp -v /etc/profile ${T}/etc/profile

list=$(ldd /bin/ls | egrep -o '/lib.*\.[0-9]')
for i in $(echo $list | awk -F '\n' '{print $1}'); do
  mkdir -p $(dirname "${T}${i}") && cp -v "$i" "${T}${i}";
done

list=$(ldd /bin/bash | egrep -o '/lib.*\.[0-9]')
for i in $(echo $list | awk -F '\n' '{print $1}'); do
  mkdir -p $(dirname "${T}${i}") && cp -v "$i" "${T}${i}";
done
```

与上一个实验不同的是，现在期望容器内的文件系统与宿主机独立。即，使用不同的根目录。因此，在代码层面需要将之前的 `mount(2)` 系统调用，改变为 `chroot(2)` 系统调用。

``` bash
$ man 2 chroot
CHROOT(2)                           Linux Programmer's Manual                           CHROOT(2)

NAME
       chroot - change root directory

SYNOPSIS
       #include <unistd.h>

       int chroot(const char *path);

DESCRIPTION
       chroot() changes the root directory of the calling process to that specified in path. This
       directory will be used for pathnames beginning with /. The root directory is inherited by
       all children of the calling process.

       Only a privileged process (Linux: one with the CAP_SYS_CHROOT capability in its user namespace)
       may call chroot().
```

最终，函数 `container_main` 的代码为:

``` c
int container_main(void *arg) {
    printf("Container - inside the container!\n");
    int rev = chroot("./root");
    if (0 != rev) {
        perror("failed to chroot");
        return 2;
    }
    rev = chdir("/");
    if (0 != rev) {
        perror("failed to chdir");
        return 3;
    }
    rev = execv(container_args[0], container_args);
    if (0 != rev) {
        perror("failed to exec");
        return 5;
    }
    printf("Something's wrong!\n");
    return 1;
}
```

与之前一样，编译并执行程序，即可进入到容器内部:

``` bash
$ gcc main.c -o mn; sudo ./mn
Parent - start a container!
Container - inside the container!
bash-4.4#
```

然后，执行 `ls` 可以看到根目录 `/` 的文件就是之前在 `root` 目录预先准备好的文件:

``` bash
bash-4.4# ls /
bin  etc  lib  lib64  usr
bash-4.4# ls /bin
bash  ls
bash-4.4#
```

综上所述，如果在 `root` 目录中保存的是一个完成的操作系统，那么，就可以实现容器内的进程就可以使用内部的 `/bin`，`/lib` 的系统环境，从而与宿主机，与其他容器相互隔离的目的。

### 备注

挂载 `tmpfs` 实验的运行环境为:

``` bash
$ cat /etc/os-release
NAME="Arch Linux"
PRETTY_NAME="Arch Linux"
ID=arch
BUILD_ID=rolling
ANSI_COLOR="38;2;23;147;209"
HOME_URL="https://www.archlinux.org/"
DOCUMENTATION_URL="https://wiki.archlinux.org/"
SUPPORT_URL="https://bbs.archlinux.org/"
BUG_REPORT_URL="https://bugs.archlinux.org/"
LOGO=archlinux

$ uname -r
4.19.104-microsoft-standard
```

`chroot` 实验的运行环境为:

``` bash
$ cat /etc/os-release
NAME="Ubuntu"
VERSION="18.04.4 LTS (Bionic Beaver)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 18.04.4 LTS"
VERSION_ID="18.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=bionic
UBUNTU_CODENAME=bionic

$ uname -r
4.15.0-112-generic
```

## UnionFS
