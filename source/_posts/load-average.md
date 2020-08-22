---
title: CPU 平均工作负责
date: 2020-08-22 22:48:39
tags:
  - linux performance
  - cpu performance
---
当程序的性能未达到预期时，大多数开发者会选择使用 `top` 指令查看指标 `CPU` 使用率。其实，还有另一个有关 `CPU` 的重要指标: 平均负责 `(load average)`。

可以使用 `top` 指令，或者 `uptime` 指令查看:

``` bash
$top

top - 23:11:02 up 19 days, 12:05,  1 user,  load average: 1.25, 1.42, 1.29
Tasks: 258 total,   2 running, 197 sleeping,   0 stopped,   0 zombie
%Cpu(s):  9.1 us,  6.8 sy,  0.3 ni, 78.1 id,  5.2 wa,  0.0 hi,  0.5 si,  0.0 st
KiB Mem : 15871204 total,   508880 free,  1675924 used, 13686400 buff/cache
KiB Swap:  2097148 total,  2096112 free,     1036 used. 14133480 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 4812 root      20   0  271208 155208  36668 S  14.6  1.0   1078:57 rancher
...
```

``` bash
$ uptime
 23:10:12 up 19 days, 12:05,  1 user,  load average: 1.65, 1.50, 1.31
```

可以发现，`uptime` 指令的输出与 `top` 指令输出的第一行几乎一致。具体每一部分的含义为:

* **当前时间**: 23:10:12
* **运行时间**: up 19 days, 12:05
* **登录用户数量**: 1 user
* **平均负载**: 过去 **1min**，**3min**，**5min** 的平均负载

## 什么是平均负载

首先，需要强调:

* **平均负载不等于 `CPU` 使用率！！！**
* **平均负载不等于 `CPU` 使用率！！！**
* **平均负载不等于 `CPU` 使用率！！！**

而是，系统处于 **可运行状态** 和 **不可中断状态** 的平均进程数，即 **平均活跃进程数**。与 `CPU` 使用率没有直接关系。

* **可运行状态**: 正在使用 `CPU` 或者正在等待 `CPU` 的进程。即 `ps` 指令显示为 `R` 状态 `(Running/Runable)` 的进程。
* **不可中断状态**: 处于内核态关键流程中的进程，且不可打断。比如最常见的是等待硬件设备的 `I/O` 响应，即 `ps` 指令中中看到的 `D` 状态 `(Uninterruptible Sleep/Disk Sleep)` 的进程。

比如，当一个进程向磁盘读写数据时，为了保证数据的一致性，在得到磁盘回复前，它是不能被其他 **进程** 或者 **中断** 打断的，这个时候的进程就处于不可中断状态。如果此时的进程被打断了，就容易出现磁盘数据与进程数据不一致的问题。

所以，**不可中断状态实际上是系统对进程和硬件设备的一种保护机制**。

## 评估平均负载

可以简单的认为，平均负载就是活跃的进程数。所以，最理想的场景是每个 `CPU` 都只需要处理一个进程，即 `CPU` 数量等于平均负载值。

因此，在分析系统平均负载是否合理时，首先应该知道当前系统的 `CPU` 数量。

平均负载可以简单的认为是活跃进程数:

``` bash
$ cat /proc/cpuinfo | grep -i 'model name' | wc -l
16
```

之后，就可以通过比较 `CPU` 的数量与平均负载大小关系，确认系统是否过载。

同时，结合过去 `1min`，`5min`，`15min` 的系统负载，就可以推测出系统负载的变化趋势。

当平均负载超过 `CPU` `50%` 时，可以认为负载较高；当超过 `70%` 时，就应该开发分析导致高负载的原因。同时，也应该将平均负载作为一项监控指标。只有结合历史数据来分析才更有意义。

## 案例分析

``` bash
$ sudo pacman -S stress sysstat
```