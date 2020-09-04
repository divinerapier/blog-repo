---
title: Pod 的基本概念
date: 2020-09-02 21:06:17
tags:
  - k8s
  - pod
---

前文提到，可以类比于虚拟机与应用程序之间的关系来理解 `Pod` 与 `Container` 之间的关系。这样就可以容易理解 **凡是调度、网络、存储，以及安全相关的属性，基本都是 Pod 级别的**。

这些属性有一个共同点: 描述 **机器** 这个整体，而不是里面运行的 **程序**。比如:

* 配置这个 **机器** 的网卡: `Pod` 的网络定义
* 配置这个 **机器** 的磁盘: `Pod` 的存储定义
* 配置这个 **机器** 的防火墙: `Pod` 的安全定义
* 这台 **机器** 运行在哪个服务器之上: `Pod` 的调度

## NodeSelector

`NodeSelector` 是一个供用户将 `Pod` 与 `Node` 进行绑定的字段，用法如下所示:

``` yml
apiVersion: v1
kind: Pod
...
spec:
 nodeSelector:
   disktype: ssd
```

这样的一个配置，意味着这个 `Pod` 永远只能运行在携带了 `disktype: ssd` 标签 `(Label)` 的节点上。否则，将调度失败。

## NodeName

当 `Kubernetes` 将 `Pod` 调度到某个 `Node` 上之后，会自动设置 `Pod` 的 `NodeName` 字段。即，`Kubernetes` 会认为所有已被赋值 `NodeName` 字段的 `Pod` 都是被调度过的。因此，通过用户也可以设置该字段来 **骗过** 调度器，比如在测试或者调试阶段。

## HostAliases

`HostAliases` 定义 `Pod` 的 `hosts` 文件 `(比如 /etc/hosts)` 里的内容。

比如，在这个 `Pod` 的 `YAML` 文件中设置了一组 `IP` 和 `Hostname` 的数据。

``` yml
apiVersion: v1
kind: Pod
...
spec:
  hostAliases:
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
...
```

在 `Pod` 启动后，`/etc/hosts` 文件的内容将如下所示:

``` bash
$ cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1 localhost
...
10.244.135.10 hostaliases-pod
10.1.2.3 foo.remote
10.1.2.3 bar.remote
```

其中，最下面两行记录，就是通过 `HostAliases` 字段写入的。

**特别注意**: 在 `Kubernetes` 中，强烈建议使用这种方式设置 hosts 文件里的内容。如果使用直接修改 `hosts` 文件的方式，在 `Pod` 被删除重建之后，`kubelet` 会还原被修改的内容。

## ShareProcessNamespace

**凡是跟容器的 `Linux Namespace` 相关的属性，也一定是 `Pod` 级别的**。

设计 `Pod` 的初衷，就是要让里面的容器尽可能多地共享 `Linux Namespace`，仅保留必要的隔离和限制能力。这样，`Pod` 之于 `Container` 就会更近似于虚拟机之于程序。

``` yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  shareProcessNamespace: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true
```

`shareProcessNamespace: true` 表示: `Pod` 里的 `Containers` 要共享 `PID Namespace`。

## 共享宿主机的 Namespace

* **HostPID** - 控制 Pod 中容器是否可以共享宿主上的进程 ID 空间。 注意，如果与 ptrace 相结合，这种授权可能被利用，导致向容器外的特权逃逸 (默认情况下 ptrace 是被禁止的)。
* **HostIPC** - 控制 Pod 容器是否可共享宿主上的 IPC 名字空间。
* **HostNetwork** - 控制是否 Pod 可以使用节点的网络名字空间。 此类授权将允许 Pod 访问本地回路 (loopback) 设备、在本地主机 (localhost) 上监听的服务、还可能用来监听同一节点上其他 Pod 的网络活动。
* **HostPorts** -提供可以在宿主网络名字空间中可使用的端口范围列表。 该属性定义为一组 HostPortRange 对象的列表，每个对象中包含 min(含) 与 max(含) 值的设置。 默认不允许访问宿主端口。

例如:

``` yml

apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  hostNetwork: true
  hostIPC: true
  hostPID: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true
```

## Lifecycle

`Container Lifecycle Hooks`。顾名思义，是在容器状态发生变化时触发一系列 **钩子**。我们来看这样一个例子:

``` yml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/usr/sbin/nginx","-s","quit"]
```

* **PostStart**: 这个回调在创建容器之后立即执行。 但是，不能保证回调会在容器入口点 `(ENTRYPOINT)` 之前执行。没有参数传递给处理程序。

* **PreStop**: 在容器因 API 请求或者管理事件 **(诸如存活态探针失败、资源抢占、资源竞争等)** 而被终止之前，此回调会被调用。如果容器已经处于终止或者完成状态，则对 preStop 回调的调用将失败。此调用是阻塞的，也是同步调用，因此必须在删除容器的调用之前完成。没有参数传递给处理程序。

> `Kubernetes` 只有在 `Pod` 结束 **(Terminated)** 的时候才会发送 preStop 事件，这意味着在 Pod 完成 **(Completed)** 时 preStop 的事件处理逻辑不会被触发。这个限制在 [issue #55087](https://github.com/kubernetes/kubernetes/issues/55807) 中被追踪。

有关终止行为的更详细描述，请参见 [终止 Pod](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-lifecycle/#termination-of-pods)。

## Status

Pending。Pod 已被 Kubernetes 系统接受 **(YAML 文件已经提交给了 Kubernetes)**，但有一个或者多个容器尚未创建亦未运行。此阶段包括等待 Pod 被调度的时间和通过网络下载镜像的时间，

Running。Pod 已经绑定到了某个节点，Pod 中所有的容器都已被创建。至少有一个容器仍在运行，或者正处于启动或重启状态。

Succeeded。Pod 中的所有容器都已成功终止，并且不会再重启。这种情况在运行一次性任务时最为常见。

Failed。Pod 中的所有容器都已终止，并且至少有一个容器是因为失败终止。也就是说，容器以非 `0` 状态退出或者被系统终止。这个状态的出现，意味着你得想办法 Debug 这个容器的应用，比如查看 Pod 的 Events 和日志。

Unknown。这是一个异常状态，意味着 Pod 的状态不能持续地被 kubelet 汇报给 kube-apiserver，这很有可能是主从节点(Master 和 Kubelet)间的通信出现了问题。

## 参考文档

* [在 Pod 中的容器之间共享进程命名空间](https://kubernetes.io/zh/docs/tasks/configure-pod-container/share-process-namespace/)
* [Share Process Namespace between Containers in a Pod](https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/)
* [Pod 安全策略](https://kubernetes.io/zh/docs/concepts/policy/pod-security-policy/)
* [Pod Security Policies](https://kubernetes.io/docs/concepts/policy/pod-security-policy/)
* [Pod 的生命周期](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-lifecycle/)
* [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
* [为容器的生命周期事件设置处理函数](https://kubernetes.io/zh/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/)
* [容器生命周期回调](https://kubernetes.io/zh/docs/concepts/containers/container-lifecycle-hooks/)
