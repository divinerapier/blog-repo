---
title: Nvidia Docker
date: 2020-11-14 18:21:01
tags:
  - gpu
  - nvidia
  - docker
---


![nvidia-container-toolkit.png](/images/using-nvidia-gpu-on-kubernetes/nvidia-container-toolkit.png)

Nvidia Container Toolkit 包含容器运行时库和一些工具，用于自动配置容器使用 GPU 资源。并且，支持多种不同的容器引擎，如 Docker、LXC、Podman 等。用户根据需要可以自行选择使用哪种引擎。

## The Architecture Overview of Nvidia Container Toolkit

Nvidia Container Toolkit 的架构允许其支持任何容器运行时。若以 Docker 为例，其由以下组件，以从上到下的层次结构组成:

* nvidia-docker2
* nvidia-container-runtime
* nvidia-container-toolkit
* libnvidia-container

下图为各个组件的关系:

![nvidia-docker-arch.png](/images/using-nvidia-gpu-on-kubernetes/nvidia-docker-arch.png)

### Components and Packages

#### libnvidia-container

提供库与 CLI 程序，实现自动化配置 GNU/Linux 容器使用 NVIDIA GPU 资源，其实现依赖于内核基础功能，且在设计上与容器运行时解耦。

libnvidia-container 提供了一个定义良好的 API 和一个封装好的 CLI 程序(nvidia-container-cli)，任何容器运行时都可以调用它来支持 NVIDIA GPU。

#### nvidia-container-toolkit

实现了 runC prestart hook 需要的接口的脚本。该脚本在容器被创建之后，启动之前被 runC 调用，且被赋予访问与容器相关联的 config.json 的权限。脚本根据 config.json 中的信息作为合适的命令行参数 (an appropriate set of flags) 来调用 libnvidia-container CLI。其中，“指定哪些 GPU 设备在容器中使用” 是最重要的参数。

该组件之前的名字是 nvidia-container-runtime-hook，现在系统上的 nvidia-container-runtime-hook 是 nvidia-container-toolkit 的符号链接。

#### nvidia-container-runtime

曾经，nvidia-container-runtime 以 runC 作为基础，添加了 NVIDIA 特定的代码。2019 年，更改为对宿主机上原生 runC 做简单的封装。nvidia-container-runtime 将 runC spec 作为输入，将 nvidia-container-toolkit 脚本作为 prestart hook 注入到 runC spec 中。然后，将修改后的带有该 hook set 的 runC spec 传递给原生 runC 并调用 runC。需要注意的是，该组件不一定是针对 docker 的(但它是针对runC的)。

当该 package 完成安装后，Docker 的 daemon.json 文件会被更新为指向这个二进制文件:

``` bash
$ cat /etc/docker/daemon.json
{
"default-runtime": "nvidia",
"runtimes": {
    "nvidia": {
        "path": "/usr/bin/nvidia-container-runtime",
        "runtimeArgs": []
    }
}
```

#### nvidia-docker2

这个 package 是架构中唯一的 docker 专用包。它采用与 nvidia-container-runtime 相关的脚本，并将其安装到 docker 的 /etc/docker/daemon.json 文件中。这样，使用者就可以运行 **docker run --runtime=nvidia ...** 来自动为容器添加对 GPU 的支持。这个 package 还安装了一个封装了原生 docker CLI 的脚本，名为 nvidia-docker，避免每次都指定 --runtime=nvidia 来调用 docker。它还允许用户在宿主机上设置环境变量 NV_GPU 来指定将哪些 GPU 注入到容器中。

## Installation

### Pre-Requisites

* [NVIDIA Drivers](https://www.nvidia.com/Download/index.aspx?lang=en-us)
* Platform Requirements:
  1. GNU/Linux x86_64 with kernel version > 3.10
  1. Docker >= 19.03 (recommended, but some distributions may include older versions of Docker. The minimum supported version is 1.12)
  1. NVIDIA GPU with Architecture > Fermi (or compute capability 2.1)
  1. NVIDIA drivers ~= 361.93 (untested on older versions)
* Docker CE

### Setting up NVIDIA Container Toolkit

安装软件源与 GPG key:

``` bash
$ distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
   && curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add - \
   && curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
```

安装 nvidia-docker2 并重启 Docker Daemon:

``` bash
$ sudo apt-get update \
   && sudo apt-get install -y nvidia-docker2 \
   && sudo systemctl restart docker
```

启动容器测试，如果得到类似如下的输出则安装成功:

``` bash
$ sudo docker run --rm --gpus all nvidia/cuda:11.0-base nvidia-smi

+-----------------------------------------------------------------------------+
| NVIDIA-SMI 450.51.06    Driver Version: 450.51.06    CUDA Version: 11.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Tesla T4            On   | 00000000:00:1E.0 Off |                    0 |
| N/A   34C    P8     9W /  70W |      0MiB / 15109MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

## 参考文档

* [NVIDIA Cloud Native Technologies](https://docs.nvidia.com/datacenter/cloud-native/index.html)
* [Container Toolkit Installation Guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)
