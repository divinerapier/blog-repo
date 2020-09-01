---
title: Pod 的意义
date: 2020-08-31 16:34:52
tags:
  - kubernetes
  - pod
---

## 容器的局限性

既然要讨论 `Pod` 的意义，或许可以想象一个如果没有 `Pod` 会是什么样子。

假设，有如下需求: 使用容器化来部署一个支持自动更新数据的文件服务。可以使用如下方法解决:

1. 在一个容器中，同时运行两个进程，一个进程提供文件服务，一个进程定时更新数据。
    * 优点:
        1. 无论是编码，还是部署都十分的容易，就像在物理机或者虚拟机里面一样
    * 缺点:
        1. 容器的本质是进程，对操作系统而言是那个由 `entrypoint` 指定的进程，其他进程都是其子进程，或子进程的子进程等。而且，`entrypoint` 没有回收僵尸进程的能力。
        2. 耦合严重
1. 每个进程分别位于各自的容器，两个容器之间通过挂载相同的外部 `volume` 实现容器间共享文件系统。
    * 优点:
        1. 将两个容器解耦，保证功能单一性
    * 缺点:
        1. 通过共享外部 `volume` 的方式无法保证数据安全性，其他容器，或宿主机也能直接访问 `volume`
1. 在上一个方法的基础上，通过 `--volume-from` 指定从其他的 `container` 挂载 `volume`。
    * 优点:
        1. 避免对外直接暴露 `volume`

到此为止，一切似乎很顺利，没有在创建 `Pod` 这样一个新的概念。

但上述方案有一个局限性，`--volume-from` 的目标容器只能是本地的其他 `container`。

假设，使用 `Docker Swarm` 部署服务。为了便于叙述，做出如下定义:

* 集群中存在两个节点。**`node-0`** 剩余 `2G` 内存，**`node-1`** 剩余 `1.5G` 内存
* 两个容器 `container-0` 与 `container-1` 各需要 `1G` 内存
* 部署过程，先启动 `container-0`，后启动 `container-1`

因此，为了让两个容器能够运行在同一个 `Node` 上，需要在启动 `container-1` 时增加限制 `affinity=container-0`。

此时，如果 `container-0` 被调度到了 `node-1` 上，`node-1` 将剩余 `0.5G` 内存。接下来，调度 `container-1` 时，`Docker Swarm` 就会发现无法找到一个合适的 `Node` 来启动 `container-1`。

以上问题被称作 [Gang Scheduling](https://en.wikipedia.org/wiki/Gang_scheduling)。

而 `Kuberntes` 则通过 `Pod` 解决了这个问题。

`Pod` 是 `Kubernetes` 中最小的可调度计算单元。在处理上述问题中，如果将 `container-0` 与 `container-1` 同时包含在一个 `Pod` 中，`Kubernetes` 在调度这个 `Pod` 时就只会考虑剩余内存不小于 `2G` 的 `node-0`。

对于上面这种，相互之间直接访问文件系统、使用 `localhost` 或 `socket` 文件进行本地通信，共享某些 `Namespace` 的一组容器，称之为 **超亲密关系**。

## Pod 实现原理

`Pod` 是一个逻辑概念。其本质是一组共享了 `Network Namespace`，并且可以声明共享同一个 `Volume` 的容器。

比如，一个包含了 `A` 与 `B` 两个容器的 `Pod`，按照 [定义](https://kubernetes.io/docs/concepts/workloads/pods/#what-is-a-pod) 的描述，可以使用如下 `Docker` 命令模拟:

``` bash
# start container A
$ docker run --net=B --volumes-from=B --name=A image-A ...
```

但此时会发现，`Pod` 内的一组容器要遵循某个特定的顺序启动，容器与容器之间不再是对等关系，而是拓扑关系。

所以，在 `Kubernetes` 项目里，通过增加一个被称作 `Infra` 的容器作为 `Pod` 里第一个启动的容器。用户定义的其他容器通过 `Join Network Namespace` 的方式，关联到 `Infra` 容器。这样的组织关系，可以用下面这样一个示意图来表达:

![pause container](/images/why-do-we-need-pods/01.png)

如上图所示，这个 `Pod` 里有两个用户容器 `A` 和 `B`，还有一个 `Infra` 容器。这个特殊容器的镜像地址为 `k8s.gcr.io/pause`，源代码位于 [github](https://github.com/kubernetes/kubernetes/blob/master/build/pause/pause.c)。同时，`kubelet` 提供参数 `--pod-infra-container-image` 支持自定义镜像。

所以，当容器 `A`、`B` 加入到 `Infra` 容器的 `Network Namespace` 后，对于容器 `A`、`B` 而言:

* 可以直接使用 `localhost` 进行通信
* 看到的网络设备跟 `Infra` 容器看到的完全一样
* 一个 `Pod` 只有一个 `IP` 地址，也就是这个 `Pod` 的 `Network Namespace` 对应的 `IP` 地址
* 其他的所有网络资源，都是一个 `Pod` 一份，并且被该 `Pod` 中的所有容器共享
* `Pod` 的生命周期只跟 `Infra` 容器一致，而与容器 `A`、`B` 无关。

而对于同一个 `Pod` 里面的所有用户容器来说，它们的进出流量，也可以认为都是通过 `Infra` 容器完成的。这一点很重要，因此为 `Kubernetes` 开发网络插件时，应该重点考虑的是如何配置这个 `Pod` 的 `Network Namespace`，而不是每一个用户容器如何使用网络配置。

而且，在这这个设计模型之下，可以方便的共享 `volume`: `Kubernetes` 项目只要把所有 `volume` 的定义都设计在 `Pod` 层级即可。之后，`Pod` 内的容器再声明挂载这个 `volume` 从而达到共享 `volume` 的目的。

``` yml

apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  restartPolicy: Never
  volumes:
  - name: shared-data
    hostPath:
      path: /data
  containers:
  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
```

## 容器设计模式

`Pod` 这种 **超亲密关系** 容器的设计思想，实际上就是希望，当用户想在一个容器里运行多个功能并不相关的应用时，应该优先考虑它们是不是更应该被描述成一个 `Pod` 里的多个容器。

要理解这个概念，也很容易。只需要将 `Pod` 理解为原来的虚拟机即可。

在虚拟机时代，服务之间也存在一定的关系。有上述那样需要共享本地文件系统的关系，也有 `API Service` 与 `DB` 之间的关系。

当需要考虑是否需要将多个容器描述成一个 `Pod` 时，可以想象一下: 如果使用虚拟机部署，是否应该将这些服务部署在同一台虚拟机上。如果需要部署在一台虚拟机才能工作，那么就需要描述为一个 `Pod`。

## 参考文档

* [Kuberntes Documentation: Pod](https://kubernetes.io/docs/concepts/workloads/pods/)
* [为什么说容器是单进程模型](https://cloud.tencent.com/developer/article/1513369)
* [Use “affinity filter” in “docker service create”](https://forums.docker.com/t/use-affinity-filter-in-docker-service-create/78402)
* [The Almighty Pause Container](https://www.ianlewis.org/en/almighty-pause-container)
* [Podman: Managing pods and containers in a local container runtime](https://developers.redhat.com/blog/2019/01/15/podman-managing-containers-pods/)
* [Design patterns for container-based distributed systems](https://www.usenix.org/system/files/conference/hotcloud16/hotcloud16_burns.pdf)
