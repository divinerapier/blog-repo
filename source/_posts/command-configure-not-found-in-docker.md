---
title: Docker 中无法找到命令 configure
date: 2020-10-18 21:02:27
tags:
  - docker
---

通过进入基于 `ubuntu:20.04` 镜像运行的容器中安装 **openmpi** 的一系列指令得到了如下 **dockerfile** 片段:

``` dockerfile
RUN wget https://download.open-mpi.org/release/open-mpi/v4.0/openmpi-4.0.5.tar.gz
RUN tar xzf openmpi-4.0.5.tar.gz -C /tmp
RUN cd /tmp/openmpi-4.0.5
RUN ./configure --with-threads=posix --enable-mpi-thread-multiple
```

以上命令可以在容器中逐条执行，但却在构建镜像时失败:

``` bash
/bin/sh: 1: ./configure: not found
```

提示 **configure** 不存在。

修复办法是将 **configure** 命令与前一条命令合并:

``` dockerfile
RUN wget https://download.open-mpi.org/release/open-mpi/v4.0/openmpi-4.0.5.tar.gz
RUN tar xzf openmpi-4.0.5.tar.gz -C /tmp
RUN cd /tmp/openmpi-4.0.5 \
  && ./configure --with-threads=posix --enable-mpi-thread-multiple \
  && make -j \
  && make -j install
```

推测原因: **docker** 镜像的每一层基于不同的 **workdir**。
