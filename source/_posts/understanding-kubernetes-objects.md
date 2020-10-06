---
title: 理解 Kubernetes 对象
date: 2020-09-30 17:19:21
tags:
  - kubernetes
---

在 Kubernetes 系统中，Kubernetes 对象 是持久化的实体。 Kubernetes 使用这些实体去表示整个集群的状态。特别地，它们描述了如下信息：

* 哪些容器化应用在运行（以及在哪些节点上）
* 可以被应用使用的资源
* 关于应用运行时表现的策略，比如重启策略、升级策略，以及容错策略

Kubernetes 对象是 **目标性记录**，即一旦创建对象，Kubernetes 系统将持续工作以确保对象存在。 通过创建对象，本质上是在告知 Kubernetes 系统，所需要的集群工作负载看起来是什么样子的， 这就是 Kubernetes 集群的 **期望状态（Desired State）**。

操作 Kubernetes 对象 —— 无论是创建、修改，或者删除 —— 需要使用 [Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api)。 比如，当使用 kubectl 命令行接口时，CLI 会执行必要的 Kubernetes API 调用， 也可以在程序中使用 [客户端库](https://kubernetes.io/docs/reference/using-api/client-libraries/) 直接调用 Kubernetes API。

## 对象规约（Spec）与状态（Status）

几乎每个 Kubernetes 对象包含两个嵌套的对象字段，它们负责管理对象的配置： 对象 spec（规约） 和 对象 status（状态） 。 对于具有 spec 的对象，你必须在创建对象时设置其内容，描述你希望对象所具有的特征： 期望状态（Desired State） 。

status 描述了对象的 当前状态（Current State），它是由 Kubernetes 系统和组件 设置并更新的。在任何时刻，Kubernetes [控制平面](https://kubernetes.io/docs/reference/glossary/?all=true#term-control-plane) 都一直积极地管理着对象的实际状态，以使之与期望状态相匹配。

例如，Kubernetes 中的 Deployment 对象能够表示运行在集群中的应用。 当创建 Deployment 时，可能需要设置 Deployment 的 spec，以指定该应用需要有 3 个副本运行。 Kubernetes 系统读取 Deployment 规约，并启动我们所期望的应用的 3 个实例 —— 更新状态以与规约相匹配。 如果这些实例中有的失败了（一种状态变更），Kubernetes 系统通过执行修正操作 来响应规约和状态间的不一致 —— 在这里意味着它会启动一个新的实例来替换。

关于对象 spec、status 和 metadata 的更多信息，可参阅 [Kubernetes API 约定](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md)。

## 描述 Kubernetes 对象

创建 Kubernetes 对象时，必须提供对象的规约，用来描述该对象的期望状态， 以及关于对象的一些基本信息（例如名称）。 当使用 Kubernetes API 创建对象时（或者直接创建，或者基于kubectl）， API 请求必须在请求体中包含 JSON 格式的信息。 **大多数情况下，需要在 .yaml 文件中为 kubectl 提供这些信息**。 kubectl 在发起 API 请求时，将这些信息转换成 JSON 格式。

这里有一个 .yaml 示例文件，展示了 Kubernetes Deployment 的必需字段和对象规约：

``` yaml
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

使用类似于上面的 .yaml 文件来创建 Deployment的一种方式是使用 kubectl 命令行接口（CLI）中的 kubectl apply 命令， 将 .yaml 文件作为参数。下面是一个示例：

``` bash
kubectl apply -f https://k8s.io/examples/application/deployment.yaml --record
```

输出类似如下这样：

``` text
deployment.apps/nginx-deployment created
```

### 必需字段

在想要创建的 Kubernetes 对象对应的 .yaml 文件中，需要配置如下的字段：

* apiVersion - 创建该对象所使用的 Kubernetes API 的版本
* kind - 想要创建的对象的类别
* metadata - 帮助唯一性标识对象的一些数据，包括一个 name 字符串、UID 和可选的 namespace

你也需要提供对象的 spec 字段。 对象 spec 的精确格式对每个 Kubernetes 对象来说是不同的，包含了特定于该对象的嵌套字段。 [Kubernetes API 参考](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/) 能够帮助我们找到任何我们想创建的对象的 spec 格式。 例如，可以从 [core/v1 PodSpec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#podspec-v1-core) 查看 Pod 的 spec 格式， 并且可以从 [apps/v1 DeploymentSpec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#deploymentspec-v1-apps) 查看 Deployment 的 spec 格式。

## 参考资料

* [理解 Kubernetes 对象](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)
* [控制平面](https://kubernetes.io/docs/reference/glossary/?all=true#term-control-plane)
