---
title: 在 Docker 中使用 OpenMPI
date: 2020-10-24 16:57:06
tags:
---

## 构建 Base 镜像

基于 **Ubuntu 20.04 + OpenMPI 4.0.5** 构建 **Base** 镜像:

``` dockerfile
FROM ubuntu:20.04

RUN apt update

RUN apt install -y wget gcc g++ make

# Install OpenSSH for MPI to communicate between containers
RUN apt-get install -y --no-install-recommends openssh-client openssh-server && \
    mkdir -p /var/run/sshd

# Allow OpenSSH to talk to containers without asking for confirmation
RUN cat /etc/ssh/ssh_config | grep -v StrictHostKeyChecking > /etc/ssh/ssh_config.new && \
    echo "    StrictHostKeyChecking no" >> /etc/ssh/ssh_config.new && \
    mv /etc/ssh/ssh_config.new /etc/ssh/ssh_config

# Install Open MPI
RUN mkdir /tmp/openmpi && \
    cd /tmp/openmpi && \
    wget https://www.open-mpi.org/software/ompi/v4.0/downloads/openmpi-4.0.5.tar.gz && \
    tar zxf openmpi-4.0.5.tar.gz && \
    cd openmpi-4.0.5 && \
    ./configure --enable-orterun-prefix-by-default && \
    make -j $(nproc) all && \
    make install && \
    ldconfig
```

### 发布镜像

``` bash
docker build -t divinerapier/openmpi:4.0.5 -f=./dockerfile .
docker push divinerapier/openmpi:4.0.5
```

## 应用程序

### 程序代码

``` c
#include <mpi.h>
#include <stdio.h>

int main(int argc, char** argv) {
    // 初始化 MPI 环境
    MPI_Init(NULL, NULL);

    // 通过调用以下方法来得到所有可以工作的进程数量
    int world_size;
    MPI_Comm_size(MPI_COMM_WORLD, &world_size);

    // 得到当前进程的秩
    int world_rank;
    MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);

    // 得到当前进程的名字
    char processor_name[MPI_MAX_PROCESSOR_NAME];
    int name_len;
    MPI_Get_processor_name(processor_name, &name_len);

    // 打印一条带有当前进程名字，秩以及
    // 整个 communicator 的大小的 hello world 消息。
    printf("Hello world from processor %s, rank %d out of %d processors\n",
           processor_name, world_rank, world_size);

    // 释放 MPI 的一些资源
    MPI_Finalize();
}
```

``` makefile
MPICC ?= mpicc

all: build

build: main.c
    $(MPICC) -o mpi_hello_world main.c

clean:
    rm -rf mpi_hello_world
```

``` dockerfile
FROM divinerapier/openmpi:4.0.5

COPY ./ /helloworld

RUN cd /helloworld \
  && make
```

### 运行应用程序

``` bash
docker build -t divinerapier/openmpi-helloworld:0.0.1 -f=./dockerfile .
docker run --rm -it divinerapier/openmpi-helloworld:0.0.1 mpirun -allow-run-as-root -np 4 /helloworld/mpi_hello_world
# Hello world from processor fa74677bda6b, rank 0 out of 4 processors
# Hello world from processor fa74677bda6b, rank 1 out of 4 processors
# Hello world from processor fa74677bda6b, rank 2 out of 4 processors
# Hello world from processor fa74677bda6b, rank 3 out of 4 processors
```
