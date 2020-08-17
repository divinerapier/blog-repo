---
title: 容器 - 隔离与限制
date: 2020-08-17 20:25:09
tags:
  - container
  - namespace
  - cgroup
---

之前提到过，容器技术就是一种沙盒技术，可以将应用及相关配置、脚本 **装** 到一个 **箱子** 中，这样，应用与应用之间就会因为有了边界而不至于相互干扰。而应用被装进箱子后，也可以被方便地搬来搬去。

但是，`Talk is cheap, show me the code!`

## 进程

如果要实现一个程序，从一个文件读取两个整数，将计算结果写入到另一个文件，则至少需要有三个文件

* 可执行文件
* 输入文件
* 输出文件

由于计算机只认识 `0` 和 `1`，因此无论用哪种语言编写这段代码，最后这三个文件都需要通过某种方式翻译成二进制文件，才能在计算机操作系统中运行与使用。初始状态时，三个文件都存放在磁盘上。可执行文件被称作程序，剩余两个文件是数据。

要完成功能，需要在计算机上执行这个程序。

首先，操作系统将 **程序** 载入到内存中，表现为指令序列。在执行过程中，当执行到从文件加载输入数据的指令时，操作系统控制存储控制器完成将数据从磁盘载入到内存。之后，操作系统读取到计算加法的指令时，通过 `CPU`、寄存器与内存的共同协作完成加法计算将计算结果暂存在内存中。最后，执行将结果保存到文件的指令时，操作系统会通过存储控制器，将内存中的结果写入到磁盘上。同时，操作系统中还需要维护其他状态，辅助这一过程顺利进行。

在操作系统中，将上述过程所涉及到的总和称作：进程。

而容器技术的核心功能，就是通过约束和修改进程的动态表现，从而为其创造出一个 **边界**。

对于 `Docker` 等大多数 `Linux` 容器来说，使用 `cgroups` 技术来制造约束，使用 `namespace` 技术来修改进程视图。

## 从创建一个容器开始

首先创建一个容器：

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

* -i: --interactive     Keep STDIN open even if not attached
* -t, --tty             Allocate a pseudo-TTY

结果就是，在操作系统中创建了一个容器，该容器中执行的程序为 `/bin/sh`。并且，在容器启动之后，申请了一个随机的 `tty`，使用交互方式访问这个容器。

此时，如果执行 `ps` 指令，会发现很有趣

``` bash
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    6 root      0:00 ps
```

在创建容器时指定的 `/bin/sh` 进程，竟然就是容器内部的第一个进程`(pid=1)`。
