---
title: Kubernetes 控制器
date: 2020-10-06 13:13:35
tags:
  - kubernetes
  - controllers
---

在机器人技术和自动化领域，控制回路（Control Loop）是一个非终止回路，用于调节系统状态。

这是一个控制环的例子：房间里的温度自动调节器。

当你设置了温度，告诉了温度自动调节器你的期望状态（Desired State）。 房间的实际温度是当前状态（Current State）。 通过对设备的开关控制，温度自动调节器让其当前状态接近期望状态。

控制器通过 [apiserver](https://kubernetes.io/docs/reference/generated/kube-apiserver/) 监控集群的公共状态，并致力于将当前状态转变为期望的状态。

## 控制器模式

一个控制器至少追踪一种类型的 Kubernetes 资源。这些 [对象](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/) 有一个代表期望状态的 spec 字段。 该资源的控制器负责确保其当前状态接近期望状态。

控制器可能会自行执行操作；在 Kubernetes 中更常见的是一个控制器会发送信息给 [API 服务器](https://kubernetes.io/docs/reference/generated/kube-apiserver/)，这会有副作用。 具体可参看后文的例子。

### 通过 API 服务器来控制

[Job](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion) 控制器是一个 Kubernetes 内置控制器的例子。 内置控制器通过和集群 API 服务器交互来管理状态。

Job 是一种 Kubernetes 资源，它运行一个或者多个 Pod， 来执行一个任务然后停止。 （一旦被调度了，对 kubelet 来说 Pod 对象就会变成了期望状态的一部分）。

在集群中，当 Job 控制器拿到新任务时，它会保证一组 Node 节点上的 kubelet 可以运行正确数量的 Pod 来完成工作。 Job 控制器不会自己运行任何的 Pod 或者容器。Job 控制器是通知 API 服务器来创建或者移除 Pod。[控制面](https://kubernetes.io/docs/reference/glossary/?all=true#term-control-plane)中的其它组件 根据新的消息作出反应（调度并运行新 Pod）并且最终完成工作。

创建新 Job 后，所期望的状态就是完成这个 Job。Job 控制器会让 Job 的当前状态不断接近期望状态：创建为 Job 要完成工作所需要的 Pod，使 Job 的状态接近完成。

控制器也会更新配置对象。例如：一旦 Job 的工作完成了，Job 控制器会更新 Job 对象的状态为 Finished。

（这有点像温度自动调节器关闭了一个灯，以此来告诉你房间的温度现在到你设定的值了）。

### 直接控制

相比 Job 控制器，有些控制器需要对集群外的一些东西进行修改。

例如，如果你使用一个控制环来保证集群中有足够的[节点](https://kubernetes.io/docs/concepts/architecture/nodes/)，那么控制就需要当前集群外的一些服务在需要时创建新节点。

和外部状态交互的控制器从 API 服务器获取到它想要的状态，然后直接和外部系统进行通信并使当前状态更接近期望状态。

（实际上有一个控制器可以水平地扩展集群中的节点。请参阅 [集群自动扩缩容](https://kubernetes.io/docs/tasks/administer-cluster/cluster-management/#cluster-autoscaling)）。

## 设计

作为设计原则之一，Kubernetes 使用了很多控制器，每个控制器管理集群状态的一个特定方面。 最常见的一个特定的控制器使用一种类型的资源作为它的期望状态， 控制器管理控制另外一种类型的资源向它的期望状态演化。

使用简单的控制器而不是一组相互连接的单体控制回路是很有用的。 控制器会失败，所以 Kubernetes 的设计正是考虑到了这一点。

> 说明：
> 可以有多个控制器来创建或者更新相同类型的对象。 在后台，Kubernetes 控制器确保它们只关心与其控制资源相关联的资源。
>
> 例如，你可以创建 Deployment 和 Job；它们都可以创建 Pod。 Job 控制器不会删除 Deployment 所创建的 Pod，因为有信息 （[标签](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)）让控制器可以区分这些 Pod。

## 运行控制器的方式

Kubernetes 内置一组控制器，运行在 [kube-controller-manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/) 内。 这些内置的控制器提供了重要的核心功能。

Deployment 控制器和 Job 控制器是 Kubernetes 内置控制器的典型例子。 Kubernetes 允许你运行一个稳定的控制平面，这样即使某些内置控制器失败了， 控制平面的其他部分会接替它们的工作。

你会遇到某些控制器运行在控制面之外，用以扩展 Kubernetes。 或者，如果你愿意，你也可以自己编写新控制器。 你可以以一组 Pod 来运行你的控制器，或者运行在 Kubernetes 之外。 最合适的方案取决于控制器所要执行的功能是什么。

## 举例说明

以 **Deployment** 为例:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

该 Deployment 定义的编排动作要求: 确保携带了 **app=nginx** 标签的 Pod 的个数，永远等于 **spec.replicas** 指定的个数，即 **2** 个。

集群会根据携带 **app=nginx** 标签的 **Pod** 的实际数量来执行创建或者删除 Pod 操作，使数量收敛于 **2**。

这时，你也许就会好奇：究竟是 Kubernetes 项目中的哪个组件，在执行这些操作呢？

在上一小节提到的 **kube-controller-manager** 就是负责管理 **Controllers** 的服务组件。并且，**Kubernetes** 项目已经包含了若干的 **Controllers**:

``` bash
$ cd kubernetes/pkg/controller/
$ ls -d */
deployment/             job/                    podautoscaler/
cloud/                  disruption/             namespace/
replicaset/             serviceaccount/         volume/
cronjob/                garbagecollector/       nodelifecycle/          replication/            statefulset/            daemon/
```

### 控制循环

正如本文开篇所说，**Kubernetes** 的 **Controllers** 遵循 **控制回路（Control Loop）** 的工作模式。

比如，对于编排的对象 X，可以用一段 Go 语言风格的伪代码，来描述其 Controller 的控制循环：

``` go
for {
    实际状态 := 获取集群中对象X的实际状态（Actual State）
    期望状态 := 获取集群中对象X的期望状态（Desired State）
    if 实际状态 == 期望状态{
        什么都不做
    } else {
        执行编排动作，将实际状态调整为期望状态
    }
}
```

一般情况:

* **实际状态来自于 Kubernetes 集群本身**: 比如，kubelet 通过心跳汇报的容器状态和节点状态，或者监控系统中保存的应用监控数据，或者控制器主动收集的它自己感兴趣的信息，这些都是常见的实际状态的来源。
* **期望状态来自于用户提交的 YAML 文件**: 比如，Deployment 对象中 Replicas 字段的值。很明显，这些信息往往都保存在 Etcd 中。

具体到本示例，Deployment 控制器的工作流程为:

1. 从 Etcd 中获取到所有携带了 **app: nginx** 标签的 Pod，统计其数量，作为实际状态
2. 从 Template 中获取 Deployment 对象的 **spec.replicas** 值，作为期望状态
3. 将两个状态做比较，然后根据比较结果，确定是创建 Pod，还是删除已有的 Pod

以上即为 **Kubernetes Controller** 的工作模式。

## 参考文档

* [Controllers](https://kubernetes.io/docs/concepts/architecture/controller/)
