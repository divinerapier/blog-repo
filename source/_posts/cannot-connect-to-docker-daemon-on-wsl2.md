---
title: WSL2 中无法连接 Docker 服务
date: 2020-10-15 17:21:20
tags:
  - windows
  - wsl2
  - docker
---

在 **WSL2** 环境中使用 **Docker** 遇到错误:

``` bash
$ docker ps
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

可以通过如下操作解决:

1. 在 **App and features** 中卸载 **Docker** 程序
1. 删除如下目录

    ``` text
    C:\Program Files\Docker
    C:\ProgramData\DockerDesktop
    C:\Users\[USERNAME]\.docker
    C:\Users\[USERNAME]\AppData\Local\Docker
    C:\Users\[USERNAME]\AppData\Roaming\Docker
    C:\Users\[USERNAME]\AppData\Roaming\Docker Desktop
    ```

1. 下载最 [新版本 **Docker**](https://docs.docker.com/docker-for-windows/edge-release-notes/)
1. 启动 **Docker**

到此为止，问题应该已经被解决了(至少我解决了)。

## 参考文档

* [Painless way to WSL 2 with Docker](https://codesthq.com/painless-way-to-wsl-2-with-docker/)
