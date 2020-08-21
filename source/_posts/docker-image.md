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

在 `Cloud Foundry` 时代，上云过程繁琐的根本原因在于: 本地环境与云主机环境不一致。

在狭义上，软件的依赖指编程时所使用的第三方库；广义上，依赖不止包括编程使用的第三方库，操作系统同样也是软件的依赖。

通过上面对 `Mount Namespace` 技术简单的实验，已经验证了在容器内打包一个完整的操作系统作为容器的 `rootfs` 是可行的。到目前为止，从技术的角度而言，已经解决了容器隔离的问题。但从使用角度，或者说用户体验的角度而言，每次构建一个容器，都要打包一份操作系统文件，似乎实在是不便于使用。

比如，开发者使用 `Ubuntu` 操作系统的 `ISO` 制作了一个 `rootfs`，并依次为基础，安装 `Java` 环境，进而部署 `Java` 应用。如果，另一个开发者也有同样的需求，或者同一开发者需要部署另一个 `Java` 应用，显然，最理想情况是能够复用之前已经安装了 `Java` 的 `rootfs` 环境，而不是重复一遍流程。

直观的解决办法，构建 `rootfs` 时，每执行一个有意义的操作之后，都生成一个新的 `rootfs`。之后，就可以选择一个合适的 `rootfs` 作为基础，添加新的操作构建目标 `rootfs`。

但这个方案并不完美，如果每次构建过程都产生一个新的 `rootfs`，最后将会导致系统内有极其多的 `rootfs`。

幸运的是，这个问题并不难解决。将问题一般化，基于既有 `A`，每一种操作 `F` 都可以产生唯一结果 `B`。并且，`B` 可以作为下一轮的输入。结果发现，这像极了 `git` 和区块链。

`Docker` 在设计 `Docker` 镜像时也是使用了类似的方法。他们引入了层 `(layer)` 概念。用户制作镜像的每一步操作，都会生成一个层，将 `rootfs` 从全量保存，改为了增量保存。

### 什么是 UnionFS

`Docker` 镜像的这种实现方式，依赖于一种叫做 `UnionFS` 的文件系统。

简单来讲，[UnionFS](https://de.wikipedia.org/wiki/UnionFS) 允许将多个设备文件或目录挂载到同一个目录上，将多个设备的内容作为整体对外展示，或者将一个设备文件挂载到一个已有的目录上。

比如，在 `Ubuntu 18.04` 上:

``` bash
# 准备测试目录
$ mkdir -p A/{a,x}
$ mkdir -p B/{b,x}
$ rm -rf C && mkdir -p C
```

使用联合挂载的方式，将这两个目录挂载到一个公共的目录 `C` 上:

``` bash
$ sudo mount -t aufs -o dirs=./A:./B none ./C
```

这时可以看到目录 `A` 与目录 `B` 被合并到了目录 `C` 中

``` bash
$ tree C
C
├── a
├── b
└── x

3 directories, 0 files
```

此时，如果修改 `C` 也会反应到对应的 `A` 或 `B` 中。

`Docker` 支持多种[存储驱动](https://docs.docker.com/storage/storagedriver/select-storage-driver/)，但目前默认使用的驱动为 `overlay2`。

### Overlay2

接下来通过一个例子来探索 `overlay2`。

首先，启动一个容器:

``` bash
$ docker run -d ubuntu:latest sleep 3600000
```

这个命令的含义是，使用 `ubuntu:latest` 这个 `Docker` 镜像来运行一个容器。然后，使用命令查看镜像的细节:

``` bash
$ docker image inspect ubuntu:latest
...
        "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/f34511250966eebf394cfcd377e3f3b3e226910881a87b14a9a9743bcbd30c05/diff:/var/lib/docker/overlay2/268f8bdf24c70efcb96bcfedbc22458b36d532bd1a9494f8a5989069eb849802/diff:/var/lib/docker/overlay2/286b92fb4ca407b7475db92eec9dccbd7bc279b968e9f7ca61deb13a9eee9c02/diff",
                "MergedDir": "/var/lib/docker/overlay2/df8d02faf11610c87d3ed7c92b5201902b87c468061bb495597fb2ce8d68d90e/merged",
                "UpperDir": "/var/lib/docker/overlay2/df8d02faf11610c87d3ed7c92b5201902b87c468061bb495597fb2ce8d68d90e/diff",
                "WorkDir": "/var/lib/docker/overlay2/df8d02faf11610c87d3ed7c92b5201902b87c468061bb495597fb2ce8d68d90e/work"
            },
            "Name": "overlay2"
        },
        "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:e1c75a5e0bfa094c407e411eb6cc8a159ee8b060cbd0398f1693978b4af9af10",
                "sha256:9e97312b63ff63ad98bb1f3f688fdff0721ce5111e7475b02ab652f10a4ff97d",
                "sha256:ec1817c93e7c08d27bfee063f0f1349185a558b87b2d806768af0a8fbbf5bc11",
                "sha256:05f3b67ed530c5b55f6140dfcdfb9746cdae7b76600de13275197d009086bb3d"
            ]
        },
...
```

* `RootFS`: 由于 `Docker` 的镜像为分层结构，制作镜像的每一步，都是一个层。因此，一个完整的 `Docker` 镜像包括 `image` 和 `layer`。为了解决空间，提高效率等目的， `Docker` 构建镜像时，使用了 `COW` 技术，即 `layer` 在 `image` 之间是被共享的。一个 `Image` 是由多个有先后逻辑顺序的 `Layer` 所构成，也就是 `RootFS` 中的 `Layer` ，上面是底层，下面是顶层。这个信息保存在 `/var/lib/docker/image/overlay2/imagedb/content/sha256/<image_id>` 文件中。

``` bash
$ ll /var/lib/docker/image/overlay2/layerdb/sha256/
drwx------ 2 root root 4096 Aug 16 13:48 27d46ebb54384edbc8c807984f9eb065321912422b0e6c49d6a9cd8c8b7d8ffc
drwx------ 2 root root 4096 Aug 16 13:48 8a8d1f0b34041a66f09e49bdc03e75c2190f606b0db7e08b75eb6747f7b49e11
drwx------ 2 root root 4096 Aug 16 13:48 e1c75a5e0bfa094c407e411eb6cc8a159ee8b060cbd0398f1693978b4af9af10
drwx------ 2 root root 4096 Aug 16 13:48 f1b8f74eff975ae600be0345aaac8f0a3d16680c2531ffc72f77c5e17cbfeeee
```

结果在 `RootFS.Layers` 中只找到了 `e1c75a5e0bfa094c407e411eb6cc8a159ee8b060cbd0398f1693978b4af9af10`。这是为 `Docker` 使用了 `ChainID` 的方式去保存其他的 `layer` 。

``` bash
$ echo -n "sha256:e1c75a5e0bfa094c407e411eb6cc8a159ee8b060cbd0398f1693978b4af9af10 sha256:9e97312b63ff63ad98bb1f3f688fdff0721ce5111e7475b02ab652f10a4ff97d" | sha256sum -
27d46ebb54384edbc8c807984f9eb065321912422b0e6c49d6a9cd8c8b7d8ffc  -

$ echo -n "sha256:27d46ebb54384edbc8c807984f9eb065321912422b0e6c49d6a9cd8c8b7d8ffc sha256:ec1817c93e7c08d27bfee063f0f1349185a558b87b2d806768af0a8fbbf5bc11" | sha256sum -
f1b8f74eff975ae600be0345aaac8f0a3d16680c2531ffc72f77c5e17cbfeeee  -

$ echo -n "sha256:f1b8f74eff975ae600be0345aaac8f0a3d16680c2531ffc72f77c5e17cbfeeee sha256:05f3b67ed530c5b55f6140dfcdfb9746cdae7b76600de13275197d009086bb3d" | sha256sum -
8a8d1f0b34041a66f09e49bdc03e75c2190f606b0db7e08b75eb6747f7b49e11  -
```

如此就找到了所有的 `layer`。

但是，上面的文件保存的只有元数据 `(metadata )`，还需要找到真实的 `rootfs` 保存的位置:

``` bash
$ cat /var/lib/docker/image/overlay2/layerdb/sha256/e1c75a5e0bfa094c407e411eb6cc8a159ee8b060cbd0398f1693978b4af9af10/cache-id
286b92fb4ca407b7475db92eec9dccbd7bc279b968e9f7ca61deb13a9eee9c02
```

`overlay2` 的所有 `rootfs` 都保存在 `/var/lib/docker/overlay2` 中，具体到上面的 `layer: e1c75a5e0bfa094c407e411eb6cc8a159ee8b060cbd0398f1693978b4af9af10` 的 `rootfs` 的位置就是 `/var/lib/docker/overlay2/286b92fb4ca407b7475db92eec9dccbd7bc279b968e9f7ca61deb13a9eee9c02`。

以此类推，可以找到上述的四个 `layer` 的 `rootfs`:

``` bash
$ cat /var/lib/docker/image/overlay2/layerdb/sha256/27d46ebb54384edbc8c807984f9eb065321912422b0e6c49d6a9cd8c8b7d8ffc/cache-id
268f8bdf24c70efcb96bcfedbc22458b36d532bd1a9494f8a5989069eb849802

$ cat /var/lib/docker/image/overlay2/layerdb/sha256/f1b8f74eff975ae600be0345aaac8f0a3d16680c2531ffc72f77c5e17cbfeeee/cache-id
f34511250966eebf394cfcd377e3f3b3e226910881a87b14a9a9743bcbd30c05

$ cat /var/lib/docker/image/overlay2/layerdb/sha256/8a8d1f0b34041a66f09e49bdc03e75c2190f606b0db7e08b75eb6747f7b49e11/cache-id
df8d02faf11610c87d3ed7c92b5201902b87c468061bb495597fb2ce8d68d90e
```

将他们通过 `Union mount` 的方式挂载到某个目录，就能得到容器完整的 `rootfs` 了。而且，可以观察出，上面得到的结果就是 `GraphDriver` 中的结果。

``` bash
$ ls /var/lib/docker/overlay2/286b92fb4ca407b7475db92eec9dccbd7bc279b968e9f7ca61deb13a9eee9c02
committed  diff  link

$ ls /var/lib/docker/overlay2/268f8bdf24c70efcb96bcfedbc22458b36d532bd1a9494f8a5989069eb849802
committed  diff  link  lower  work

$ ls /var/lib/docker/overlay2/f34511250966eebf394cfcd377e3f3b3e226910881a87b14a9a9743bcbd30c05
committed  diff  link  lower  work

$ ls /var/lib/docker/overlay2/df8d02faf11610c87d3ed7c92b5201902b87c468061bb495597fb2ce8d68d90e
committed  diff  link  lower  work
```

除了最底层 `286b92fb4ca407b7475db92eec9dccbd7bc279b968e9f7ca61deb13a9eee9c02` 之外，其他各上层的 `rootfs` 中都存在 `lower` 目录，这是保存各自的底层 `(文档中表述为 parent)`。比如对于倒数第二层 `268f8bdf24c70efcb96bcfedbc22458b36d532bd1a9494f8a5989069eb849802`

``` bash
$ cat /var/lib/docker/overlay2/268f8bdf24c70efcb96bcfedbc22458b36d532bd1a9494f8a5989069eb849802/lower
l/QEYZLDRIUA2DNUJIGGVCZYDVRM

$ ls -l /var/lib/docker/overlay2/l/QEYZLDRIUA2DNUJIGGVCZYDVRM
/var/lib/docker/overlay2/l/QEYZLDRIUA2DNUJIGGVCZYDVRM -> ../286b92fb4ca407b7475db92eec9dccbd7bc279b968e9f7ca61deb13a9eee9c02/diff
```

最后，根据当前的 `lower` 可以得到底层的 `diff` 。

通过上述操作，将 `lower` 与 `diff` 关联起来了:

* `lower`: 可理解为当前的镜像层，对于当前层而言，是只读的
* `diff`: 是容器可读可写层，初始为空，容器内有文件被修改时，这个文件夹就会有对应的变化，也就是所谓的 `COW`

使用如下方式验证:

``` bash
$ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
7baea70ac0a0        ubuntu:latest       "sleep 3600000"     2 hours ago         Up 2 hours                              vigorous_poitras

$ docker inspect 7baea70ac0a0
...
        "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/9667b87c5a650e82e2a27a6420813fd8b7891a0e37b13558448e6f82a2e7a877-init/diff:/var/lib/docker/overlay2/df8d02faf11610c87d3ed7c92b5201902b87c468061bb495597fb2ce8d68d90e/diff:/var/lib/docker/overlay2/f34511250966eebf394cfcd377e3f3b3e226910881a87b14a9a9743bcbd30c05/diff:/var/lib/docker/overlay2/268f8bdf24c70efcb96bcfedbc22458b36d532bd1a9494f8a5989069eb849802/diff:/var/lib/docker/overlay2/286b92fb4ca407b7475db92eec9dccbd7bc279b968e9f7ca61deb13a9eee9c02/diff",
                "MergedDir": "/var/lib/docker/overlay2/9667b87c5a650e82e2a27a6420813fd8b7891a0e37b13558448e6f82a2e7a877/merged",
                "UpperDir": "/var/lib/docker/overlay2/9667b87c5a650e82e2a27a6420813fd8b7891a0e37b13558448e6f82a2e7a877/diff",
                "WorkDir": "/var/lib/docker/overlay2/9667b87c5a650e82e2a27a6420813fd8b7891a0e37b13558448e6f82a2e7a877/work"
            },
            "Name": "overlay2"
        },
...

$ docker exec -ti 7baea70ac0a0 bash
touch ~/a.txt
```

然后在另一个终端:

``` bash
$ tree /var/lib/docker/overlay2/9667b87c5a650e82e2a27a6420813fd8b7891a0e37b13558448e6f82a2e7a877
├── diff
│   └── root
│       └── a.txt
├── link
├── lower
├── merged
└── work
    └── work
```

在 `diff` 中多出了文件 `root/a.txt`。

此外，很容易发现还有一个以 `-init` 结尾的文件，这同样是一个层，夹在只读层和读写层之间。`Init` 层是 `Docker` 项目单独生成的一个内部层，专门用来存放 `/etc/hosts`、 `/etc/resolv.conf` 等信息。

需要这样一层的原因是，这些文件本来属于只读的 `Ubuntu` 镜像的一部分，但是用户往往需要在启动容器时写入一些指定的值比如 `hostname`，所以就需要在可读写层对它们进行修改。

可是，这些修改往往只对当前的容器有效，我们并不希望执行 `docker commit` 时，把这些信息连同可读写层一起提交掉。

所以， `Docker` 做法是，在修改了这些文件之后，以一个单独的层挂载了出来。而用户执行 `docker commit` 只会提交可读写层，所以是不包含这些内容的。

最终，这几个层都被联合挂载，表现为一个完整的 `Ubuntu` 操作系统供容器使用。

## 总结

`Mount Namespace` 技术为容器提供了独立 `rootfs` 的能力，使容器在本地环境，测试环境，云环境之间具备了真正的 **一致性**。

`UnionFS` 为 `Docker` 镜像提供了快速迭代，分层下载，复用已有镜像等能力。

## 参考文档

[DOCKER基础技术：AUFS](https://coolshell.cn/articles/17061.html)
[一文读懂容器三大核心技术——Namespace，Cgroup和UnionFS](https://blog.csdn.net/ra681t58cjxsgckj31/article/details/104707642)
[Docker storage drivers](https://docs.docker.com/storage/storagedriver/select-storage-driver/)
