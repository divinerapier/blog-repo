---
title: 容器 - 隔离与限制
date: 2020-08-17 20:25:09
tags:
  - container
  - namespace
  - cgroup
---

容器技术的本质是一种沙盒技术，可以将应用及相关配置、脚本 **装** 到一个 **箱子** 中。这样，多个应用之间就会因为有了边界而避免相互干扰。并且，应用被装进箱子后，也便于移动。

## 进程

要实现一个程序，从一个文件读取两个整数，将计算结果写入到另一个文件中，则至少需要有三个文件:

* 可执行文件
* 输入文件
* 输出文件

由于计算机只认识 `0` 和 `1`，因此无论用哪种语言编写这段代码，最后这三个文件都需要通过某种方式翻译成二进制文件，才能在计算机操作系统中运行与使用。初始状态时，三个文件都存放在磁盘上。可执行文件被称作 **程序**，剩余两个文件是 **数据**。

要完成功能，需要在计算机上执行这个程序。

首先，操作系统将 **程序** 载入到内存中，表现为指令序列。在执行过程中，当执行到从文件加载输入数据的指令时，操作系统控制存储控制器完成将数据从磁盘载入到内存。之后，操作系统读取到计算加法的指令时，通过 `CPU`、寄存器与内存的共同协作完成加法计算将计算结果暂存在内存中。最后，执行将结果保存到文件的指令时，操作系统会通过存储控制器，将内存中的结果写入到磁盘上。同时，操作系统中还需要维护其他状态，辅助这一过程顺利进行。

在操作系统中，将上述过程所涉及到的总和称作: 进程。

而容器技术的核心功能，就是通过约束和修改进程的动态表现，从而为其创造出一个 **边界**。

对于 `Docker` 等大多数 `Linux` 容器来说，使用 `cgroups` 技术来制造约束，使用 `namespace` 技术来修改进程视图。

## 创建容器

首先创建一个容器:

``` bash
$ docker run --help
Usage:  docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

$ docker run -it busybox /bin/sh
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
91f30d776fb2: Pull complete
Digest: sha256:9ddee63a712cea977267342e8750ecbc60d3aab25f04ceacfa795e6fce341793
Status: Downloaded newer image for busybox:latest
/ #
```

参数 `-it` 的含义是

* **-i**: --interactive     Keep STDIN open even if not attached
* **-t**, --tty             Allocate a pseudo-TTY

结果就是，在操作系统中创建了一个容器，该容器中执行的程序为 `/bin/sh`。并且，在容器启动之后，申请了一个随机的 `tty`，使用交互方式访问这个容器。

现在，执行 `ps` 指令，结果真是 **Amazing** 啊:

``` bash
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    6 root      0:00 ps
```

在创建容器时指定的 `/bin/sh` 进程，竟然就是容器内部的第一个进程`(pid=1)`，并且，这个容器内只有两个进程正在运行中，另一个进程是正在执行的 `ps` 指令。

而这正是 `namespace` 技术的神奇之处。

## Namespace

首先，通过 `manpages` 对 `namespace` 有一个初步的了解:

``` bash
$ man namespaces
NAMESPACES(7)                       Linux Programmer's Manual                       NAMESPACES(7)

NAME
       namespaces - overview of Linux namespaces

DESCRIPTION
       A namespace wraps a global system resource in an abstraction that makes it appear to the processes
       within the namespace that they have their own isolated instance of the global resource. Changes to
       the global resource are visible to other processes that are members of the namespace, but are
       invisible to other processes. One use of namespaces is to implement containers.

       This page provides pointers to information on the various namespace types, describes the associated
      /proc files, and summarizes the APIs for working with namespaces.
       ...
```

总的来说，`Linux Namespace` 提供了一种内核级别隔离系统资源的方法，通过将系统的全局资源放在不同的 `Namespace` 中，来实现资源隔离的目的。不同 `Namespace` 的程序，可以享有一份独立的系统资源。目前 `Linux` 中提供了以下几种系统资源的隔离机制：

|Namespace| Flag            |Page                  |Isolates|
|:--------|:----------------|:---------------------|:-------|
|Cgroup    |CLONE_NEWCGROUP |cgroup_namespaces(7)  |Cgroup root directory|
|IPC       |CLONE_NEWIPC    |ipc_namespaces(7)     |System V IPC, POSIX message queues|
|Network   |CLONE_NEWNET    |network_namespaces(7) |Network devices, stacks, ports, etc.|
|Mount     |CLONE_NEWNS     |mount_namespaces(7)   |Mount points|
|PID       |CLONE_NEWPID    |pid_namespaces(7)     |Process IDs|
|Time      |CLONE_NEWTIME   |time_namespaces(7)    |Boot and monotonic clocks|
|User      |CLONE_NEWUSER   |user_namespaces(7)    |User and group IDs|
|UTS       |CLONE_NEWUTS    |uts_namespaces(7)     |Hostname and NIS domain name|

### 如何使用 Namespace 技术

以 `PID Namespace` 为例，简单说明如何在编程中使用这种技术。

`Linux` 实现 `Namespace` 机制的方式，是通过在创建进程的时候，传入特定的选项。更具体一些，就是在调用

``` bash
$ man 2 clone
CLONE(2)                            Linux Programmer's Manual                           CLONE(2)

NAME
       clone, __clone2, clone3 - create a child process

SYNOPSIS
       /* Prototype for the glibc wrapper function */

       #define _GNU_SOURCE
       #include <sched.h>

       int clone(int (*fn)(void *), void *stack, int flags, void *arg, ...
                 /* pid_t *parent_tid, void *tls, pid_t *child_tid */ );
```

系统调用时，传入对应的 `Flag` 作为参数 `flag` 的值。比如:

```  c
int pid = clone(main_function, stack_size, SIGCHLD, NULL);
```

就会创建一个新的进程，并且返回它的进程号 `pid`。

如果，同时指定 `CLONE_NEWPID` 参数:

``` c
int pid = clone(main_function, stack_size, CLONE_NEWPID | SIGCHLD, NULL);
```

新创建进程将会 **看到** 一个全新的进程空间，在这个进程空间里，它的 `pid` 是 `1`。之所以说 **看到**，是因为这只是一个 **障眼法**，在宿主机真实的进程空间里，这个进程的 `pid` 还是真实的数值，比如 `404`。

如果多次执行上面的 `clone()` 调用，就会创建多个 `PID Namespace`，而每个 `Namespace` 里的应用进程都会认为自己是当前容器里的 **第 1 号进程**，它们既看不到宿主机里真正的进程空间，也看不到其他 `PID Namespace` 里的具体情况。

而其他的几种 `Namespace`，在写法上与 `PID Namespace` 是相同的，区别只在于目的不同。

所以，`Docker` 容器这个听起来玄而又玄的概念，实际上是在创建容器进程时，指定了这个进程所需要启用的一组 `Namespace` 参数。这样，容器就只能 **看到** 当前 `Namespace` 所限定的资源、文件、设备、状态，或者配置。而对于宿主机以及其他不相关的程序，它就完全看不到了。

本质仍旧是进程。因此，在之前出现过的虚拟机对比容器的图片中，才没有出现 `Docker` 的位置。因为，容器只是通过 `Namespace` 技术被隔离的进程，与其他进程一样，也是直接运行在宿主机操作系统之上的。`Docker` 只是充当了一个管理者的身份。

![virtualization-vs-containers](/images/what-is-the-docker/virtualization-vs-containers.png)

既然，虚拟机与容器是两种不同的技术，那么二者之间就应该有一些区别。

#### 容器的优势

##### 占用资源小

容器占用的内存，要比同等功能的虚拟机占用的内存小，因为虚拟机本身也需要消耗一定的资源。比如，运行 `CentOS` 的 `KVM` 虚拟机至少需要 `100 ~ 200 MB` 的内存

##### 更好的 I/O 性能

在参考阅读 [An Updated Performance Comparison of Virtual Machines and Linux Containers](https://dominoweb.draco.res.ibm.com/reports/rc25482.pdf) 中显示

* 在随机读写场景中，无论是 `iops` 还是 `latency`，容器的性能都接近于物理机性能，远好于 `KVM` 虚拟机
* 在顺序读写场景中，容器的性能基本与物理机性能，略好于 `KVM` 虚拟机
* 在 `host network` 场景中，容器的性能接近于物理机的性能，好于 `KVM` 虚拟机
* 在 `nat network` 场景中，容器的性能略逊于 `KVM` 虚拟机，但差别不大

总的来说，容器的 `I/O` 性能优于 `KVM` 虚拟机

#### 容器的劣势

##### 平台依赖性

* 尽管，可以通过 `Mount Namespace` 的方式挂起与宿主机不同的 `Linux` 发行版，但容器中进程使用的内核依然是宿主机的内核。
  * 因此 `Windows` 耗费了那么长的时间才对容器技术有了比较好的支持。
  * 同样，也无法在低版本内核的宿主机上运行高版本内核的容器。

##### 有限的隔离

部分资源无法被隔离，比如 **时间**。

如果在容器中调用 `settimeofday(2)` 系统调用修改系统时间，会导致宿主机系统的时间被修改，这显然是一个很可怕的事情。因此，在使用容器的时候必须要了解 **什么能做，什么不能做**。

## Cgroup

`Namespace` 技术的作用是隔离进程，但只有隔离是不够的。虽然，容器内的进程无法感知到其他运行在宿主机上的进程或容器，但从操作系统的角度而言，所有的进程与容器都是平等的地位。这就意味着，虽然容器是一个独立的小空间，但其可用的资源仍旧与其他进程之间有着竞争的关系。

因此，就需要用到 `Cgroup` 技术的帮助了。

`Cgroup` 全称为 `Control Group`，用途是限制一个进程组能够使用的资源上限，包括 `CPU`、`MEM`、`DISK`、`NETWORK` 等。

### Cgroup In Action

在 `Linux` 中 `Everything is a file`，因此，操作 `Cgroup` 的途径也是通过文件系统。

``` bash
$ mount -t cgroup
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/cpu type cgroup (rw,nosuid,nodev,noexec,relatime,cpu)
cgroup on /sys/fs/cgroup/cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpuacct)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/net_cls type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_prio)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/rdma type cgroup (rw,nosuid,nodev,noexec,relatime,rdma)
```

可以看到，`Cgroup` 是一系列定义在 `/sys/fs/cgroup/` 目录下的各种文件或文件夹。每一个文件夹都被用来限制某一种特定的资源。以 `cpu` 资源为例:

``` bash
$ ls /sys/fs/cgroup/cpu
000-metadata  cgroup.clone_children  cpu.cfs_period_us  cpu.rt_runtime_us  docker    notify_on_release  systemreserved
001-binfmt    cgroup.procs           cpu.cfs_quota_us   cpu.shares         kmsg      podruntime         tasks
002-bridge    cgroup.sane_behavior   cpu.rt_period_us   cpu.stat           kubepods  release_agent
```

列举了限制 `cpu` 资源的各种参数选项。

使用 `Cgroup` 的方式非常简单，在对应的资源目录下面创建一个目录，比如:

``` bash
$ sudo mkdir -p /sys/fs/cgroup/cpu/container

$ ls /sys/fs/cgroup/cpu/container
cgroup.clone_children  cpu.cfs_period_us  cpu.rt_period_us   cpu.shares  notify_on_release
cgroup.procs           cpu.cfs_quota_us   cpu.rt_runtime_us  cpu.stat    tasks
```

这个目录被称作 **控制组**，`Linux` 会在控制组下自动创建各种资源限制文件。

然后，在后台执行脚本:

``` bash
$ while : ; do : ; done &
[1] 12555
```

显然，这是一个死循环进程，会占用 `100%` 的 `CPU`。使用 `top` 指令，也可以确认确实使用了 `100%` 的 `CPU`。

``` bash
$ top -p 12555
top - 21:00:26 up  4:43,  0 users,  load average: 0.66, 0.66, 0.56
Tasks:   1 total,   1 running,   0 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  2.5 sy,  3.7 ni, 93.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   7958.1 total,   5885.9 free,    539.2 used,   1533.0 buff/cache
MiB Swap:  16384.0 total,  16384.0 free,      0.0 used.   7183.5 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
12555 diviner+  25   5   13576   4816      0 R 100.0   0.1   0:46.30 zsh
```

接下来，是时候展现 `Cgroup` 的魔法了。首先，查看两个接下来要使用的关键文件:

``` bash
$ cat /sys/fs/cgroup/cpu/container/cpu.cfs_quota_us
-1
$ cat /sys/fs/cgroup/cpu/container/cpu.cfs_period_us
100000
```

* **cfs_period_us**: 资源组获取资源的时间周期，单位微妙
* **cfs_quota_us**: 在获取资源时间周期内获得资源的时间，单位微妙

所以 `usage = cfs_quota_us / cfs_period_us`。

为了验证这一点，尝试将进程的 `CPU` 使用率控制在 `20%`。

首先，根据上面推测的公式，因为 `cfs_period_us = 100000`，所以应该修改 `cfs_quota_us`:

``` bash
$ echo 20000 | sudo tee /sys/fs/cgroup/cpu/container/cpu.cfs_quota_us
```

但这还不够，还需要将被限制的进程的 `PID` 写入到 **控制组** `container` 的 `tasks`:

``` bash
$ echo 12555 | sudo tee /sys/fs/cgroup/cpu/container/tasks
```

然后使用 `top` 查看:

``` bash
top - 21:09:11 up  4:52,  0 users,  load average: 0.41, 0.76, 0.70
Tasks:   1 total,   1 running,   0 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  0.8 sy,  0.9 ni, 98.4 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   7958.1 total,   5883.8 free,    541.1 used,   1533.1 buff/cache
MiB Swap:  16384.0 total,  16384.0 free,      0.0 used.   7181.6 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
12555 diviner+  25   5   13576   4816      0 R  20.0   0.1   8:17.25 zsh
```

`Linux Cgroups` 的设计还是比较易用的，可以简单地理解为，一个子系统目录加上一组资源限制文件的组合。而对于 `Docker` 等 `Linux` 容器项目来说，它们只需要在每个子系统下面，为每个容器创建一个**控制组** (即创建一个新目录)，然后在启动容器进程之后，把这个进程的 `PID` 填写到对应控制组的 `tasks` 文件中就可以了。

## Cgroup in Docker

在通过 `Docker` 启动容器的时候，也可以限制 `cpu` 使用率:

``` bash
$ docker run -it --cpu-period=100000 --cpu-quota=20000 ubuntu /bin/bash
611f8f2c0c7613c2f3f3964e7dc3dbe407c74689e376736e34d7fe4735ec95d4
```

``` bash
$ cat /sys/fs/cgroup/cpu/docker/611f8f2c0c7613c2f3f3964e7dc3dbe407c74689e376736e34d7fe4735ec95d4/cpu.cfs_period_us
100000

$ cat /sys/fs/cgroup/cpu/docker/611f8f2c0c7613c2f3f3964e7dc3dbe407c74689e376736e34d7fe4735ec95d4/cpu.cfs_quota_us
20000
```

此时，这个容器最多只能使用 `20%` 的 `CPU`。

## 参考阅读

* [An Updated Performance Comparison of Virtual Machines and Linux Containers](https://dominoweb.draco.res.ibm.com/reports/rc25482.pdf)
