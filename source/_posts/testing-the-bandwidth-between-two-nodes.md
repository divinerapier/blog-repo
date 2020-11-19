---
title: 测试节点之间的网络带宽
date: 2020-11-19 13:14:30
tags:
  - network
---

昨天同事找到我，说 **nfs** 太慢了，通过 **iostat** 看只有 **1-2MB/s** 的写入速度。在通过 **fio** 测试磁盘顺序写入速度，得到结果为 **300MB/s** 之后，遂怀疑是网络的问题。

## iperf

> iperf is a tool for performing network throughput measurements.  It can test either TCP or UDP throughput.  To perform an iperf test the user must establish both a server (to discard traffic) and a client (to generate traffic).

此处省略安装过程。

### 测试网络带宽

**iperf** 通过使用不同的命令行参数，支持分别作为 **服务端** 或 **客户端**。

#### 启动服务端

监听默认端口 **5001**，启动服务端:

``` bash
iperf -s
```

或者，监听指定端口，启动服务端:

``` bash
iperf -s -p <port>
```

#### 启动客户端

连接默认端口 **5001**，启动客户端:

``` bash
iperf -c <server-host>
```

或者，连接指定端口，启动客户端:

``` bash
iperf -c <server-host> -p <port>
```

同时，**iperf** 也支持多线程的客户端:

``` bash
iperf -c <server-host> -p <port> -P <threadiness>
```

#### 测试结果

单线程客户端的测试结果:

``` bash
------------------------------------------------------------
Client connecting to 10.33.28.26, TCP port 9999
TCP window size:  170 KByte (default)
------------------------------------------------------------
[  3] local 172.29.60.164 port 37290 connected with 10.33.28.26 port 9999
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0-10.6 sec  24.0 MBytes  19.0 Mbits/sec
```

多线程客户端的测试结果，两个节点都有 **16** 个 **CPU** 核心:

``` bash
------------------------------------------------------------
Client connecting to 10.33.28.26, TCP port 9999
TCP window size: 85.0 KByte (default)
------------------------------------------------------------
[ 18] local 172.29.60.164 port 37392 connected with 10.33.28.26 port 9999
[ 17] local 172.29.60.164 port 37390 connected with 10.33.28.26 port 9999
[  4] local 172.29.60.164 port 37364 connected with 10.33.28.26 port 9999
[ 11] local 172.29.60.164 port 37378 connected with 10.33.28.26 port 9999
[  9] local 172.29.60.164 port 37374 connected with 10.33.28.26 port 9999
[  8] local 172.29.60.164 port 37372 connected with 10.33.28.26 port 9999
[ 10] local 172.29.60.164 port 37376 connected with 10.33.28.26 port 9999
[  6] local 172.29.60.164 port 37366 connected with 10.33.28.26 port 9999
[ 13] local 172.29.60.164 port 37382 connected with 10.33.28.26 port 9999
[ 14] local 172.29.60.164 port 37384 connected with 10.33.28.26 port 9999
[  3] local 172.29.60.164 port 37362 connected with 10.33.28.26 port 9999
[  5] local 172.29.60.164 port 37368 connected with 10.33.28.26 port 9999
[  7] local 172.29.60.164 port 37370 connected with 10.33.28.26 port 9999
[ 15] local 172.29.60.164 port 37386 connected with 10.33.28.26 port 9999
[ 16] local 172.29.60.164 port 37388 connected with 10.33.28.26 port 9999
[ 12] local 172.29.60.164 port 37380 connected with 10.33.28.26 port 9999
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0-10.2 sec  2.62 MBytes  2.17 Mbits/sec
[ 13]  0.0-10.2 sec  2.88 MBytes  2.36 Mbits/sec
[ 15]  0.0-10.2 sec  1.88 MBytes  1.54 Mbits/sec
[  7]  0.0-10.9 sec  1.50 MBytes  1.15 Mbits/sec
[  6]  0.0-11.0 sec  1.75 MBytes  1.34 Mbits/sec
[ 17]  0.0-11.2 sec  1.88 MBytes  1.40 Mbits/sec
[  5]  0.0-11.3 sec  1.50 MBytes  1.12 Mbits/sec
[ 10]  0.0-11.6 sec  1.88 MBytes  1.35 Mbits/sec
[  9]  0.0-12.4 sec  1.62 MBytes  1.10 Mbits/sec
[  4]  0.0-13.0 sec  2.88 MBytes  1.85 Mbits/sec
[ 11]  0.0-13.0 sec  4.62 MBytes  2.98 Mbits/sec
[ 14]  0.0-13.1 sec  4.88 MBytes  3.12 Mbits/sec
[  8]  0.0-13.9 sec  2.12 MBytes  1.28 Mbits/sec
[ 16]  0.0-14.1 sec  2.00 MBytes  1.19 Mbits/sec
[ 18]  0.0-14.1 sec  2.12 MBytes  1.26 Mbits/sec
[ 12]  0.0-14.3 sec  1.73 MBytes  1.01 Mbits/sec
[SUM]  0.0-14.3 sec  37.9 MBytes  22.1 Mbits/sec
```

## 结论

瓶颈在网络带宽，悲哀。
