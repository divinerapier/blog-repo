---
title: Kubernetes Deployment
date: 2020-10-27 10:48:23
tags:
  - kubernetes
  - controllers
---

一个 Deployment 控制器为 [Pods](https://kubernetes.io/docs/concepts/workloads/pods/) 和 [ReplicaSets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) 提供声明式的更新能力。

用户负责描述 Deployment 中的 **目标状态**，而 Deployment 控制器以受控速率更改 **实际状态**，使其变为 **期望状态**。用户可以定义 Deployment 以创建新的 ReplicaSet，或删除现有 Deployment， 并通过新的 Deployment 接收(adopt)其资源。

> **说明**： 不要管理 Deployment 所拥有的 ReplicaSet 。 如果存在下面未覆盖的使用场景，请考虑在 Kubernetes 仓库中提出 Issue。

## 用例

以下是 Deployments 的典型用例：

* [创建 Deployment 使 ReplicaSet 上线(rollout)](#Deployment)。 ReplicaSet 在后台创建 Pods。 检查 ReplicaSet 的上线状态，查看其是否成功。
* 通过更新 Deployment 的 PodTemplateSpec，声明 Pod 的新状态 。 新的 ReplicaSet 会被创建，Deployment 以受控速率将 Pod 从旧 ReplicaSet 迁移到新 ReplicaSet。 每个新的 ReplicaSet 都会更新 Deployment 的修订版本。
* 如果 Deployment 的当前状态不稳定，回滚到较早的 Deployment 版本。 每次回滚都会更新 Deployment 的修订版本。
* 扩大 Deployment 规模以承担更多负载。
* 暂停 Deployment 以应用对 PodTemplateSpec 所作的多项修改， 然后恢复其执行以启动新的上线版本。
* 使用 Deployment 状态 来判定上线过程是否出现停滞。
* 清理较旧的不再需要的 ReplicaSet。

## 创建 Deployment

下面是 Deployment 示例。Deployment 创建一个 ReplicaSet，负责启动三个 **nginx** Pods:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
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

在该例中:

* 创建名为 **nginx-deployment** (由 **.metadata.name** 字段标明) 的 Deployment。
* 该 Deployment 创建三个 (由 **.spec.replicas** 字段标明) Pod 副本。
* **.spec.selector** 字段定义 Deployment 如何查找要管理的 Pods。 在这里，你只需选择在 Pod 模板中定义的标签（app: nginx）。 不过，更复杂的选择规则是也可能的，只要 Pod 模板

    > **说明**： **matchLabels** 字段是 {key,value} 字典映射。在 **matchLabels** 映射中的单个 {key,value} 映射等效于 **matchExpressions** 中的一个元素，即其 key 字段是 “key”，operator 为 “In”，value 数组仅包含 “value”。在 **matchLabels** 和 **matchExpressions** 中给出的所有条件都必须满足才能匹配。

* **.spec.template** 字段包含以下子字段:
  * 使用 **.metadata.labels** 字段为 Pod 设置标签 **app: nginx**
  * **.template.spec** 字段表示 Pod 的模板，指示 Pods 运行一个 **nginx** 容器，该容器运行 **nginx:1.14.2** 镜像。
  * 创建一个容器，使用 **.spec.template.spec.containers[0].name** 字段 **nginx** 作为名字

开始之前，请确保的 Kubernetes 集群已启动并运行。 按照以下步骤创建上述 Deployment:

1. 通过运行以下命令创建 Deployment:

    ``` bash
    kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml
    ```

1. 检查 Deployment 是否已创建。如果仍在创建 Deployment， 则输出类似于:

    ``` bash
    kubectl get deployments

    NAME               READY   UP-TO-DATE   AVAILABLE   AGE
    nginx-deployment   0/3     0            0           1s
    ```

    在检查集群中的 Deployment 时，所显示的字段有：
    * NAME 列出了集群中 Deployment 的名称。
    * READY 显示应用程序的可用的 **副本** 数。显示的模式是“就绪个数/期望个数”。
    * UP-TO-DATE 显示为了打到期望状态已经更新的副本数。
    * AVAILABLE 显示应用可供用户使用的副本数。
    * AGE 显示应用程序运行的时间。
    请注意期望副本数是根据 **.spec.replicas** 字段设置 3。

1. 查看 Deployment 上线状态:

    ``` bash
    kubectl rollout status deployment.v1.apps/nginx-deployment

    Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
    deployment "nginx-deployment" successfully rolled out
    ```

1. 查看 Deployment 创建的 ReplicaSet (rs):

    ``` bash
    kubectl get rs

    NAME                          DESIRED   CURRENT   READY   AGE
    nginx-deployment-75675f5897   3         3         3       18s
    ```

    ReplicaSet 输出中包含以下字段:

    * NAME 列出名字空间中 ReplicaSet 的名称；
    * DESIRED 显示应用的期望副本个数，即在创建 Deployment 时所定义的值。 此为期望状态；
    * CURRENT 显示当前运行状态中的副本个数；
    * READY 显示应用中有多少副本可以为用户提供服务；
    * AGE 显示应用已经运行的时间长度。
    > **注意**: ReplicaSet 的名称始终被格式化为 **[Deployment名称]-[随机字符串]**。 其中的随机字符串是使用 **pod-template-hash** 作为种子随机生成的。

1. 查看每个 Pod 自动生成的标签:

    ``` bash
    kubectl get pods --show-labels

    NAME                                READY     STATUS    RESTARTS   AGE       LABELS
    nginx-deployment-75675f5897-7ci7o   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
    nginx-deployment-75675f5897-kzszj   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
    nginx-deployment-75675f5897-qqcnn   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
    ```

    所创建的 ReplicaSet 确保总是存在三个 **nginx** Pod。

> **说明**: 必须在 Deployment 中指定适当的 **.spec.selector** 和 **.spec.template.metadata.labels**。不要与其他控制器重叠。 Kubernetes 不会阻止这样做，但是如果多个控制器具有重叠的 selector，它们可能会发生冲突 执行难以预料的操作。

### Pod-template-hash 标签

> **注意**: 不要更改此标签

Deployment 控制器将 **pod-template-hash** 标签添加到 Deployment 所创建或接收(adopt) 的 每个 ReplicaSet 。

此标签可确保 Deployment 的子 ReplicaSets 不重叠。 标签是通过对 ReplicaSet 的 PodTemplate 进行哈希处理。 所生成的哈希值被添加到 ReplicaSet 的 **selector**、Pod 的 **label**，并存在于在 ReplicaSet 可能拥有的任何现有 Pod 中。

## 更新 Deployment

> **说明**: 仅当 Deployment 的 **.spec.template** 发生改变时，例如模板的标签或容器镜像被更新，才会触发 Deployment 上线。其他更新(如对 Deployment 执行扩缩容的操作) 不会触发上线动作。

按照以下步骤更新 Deployment:

  1. 更新 **nginx** Pod 镜像，从 **nginx:1.14.2** 到 **nginx:1.16.1**:

  ``` bash
  $ kubectl --record deployment.apps/nginx-deployment set image \
     deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1

  deployment.apps/nginx-deployment image updated
  ```

  或者使用下面的命令:

  ``` bash
  $ kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1 --record

  deployment.apps/nginx-deployment image updated
  ```

  或者，可以 **edit** Deployment 并将 **.spec.template.spec.containers[0].image** 从 **nginx:1.14.2** 更改至 **nginx:1.16.1**

  ``` bash
  $ kubectl edit deployment.v1.apps/nginx-deployment

  deployment.apps/nginx-deployment edited
  ```

  1. 查看上线状态，运行:

  ``` bash
  $ kubectl rollout status deployment.v1.apps/nginx-deployment

  Waiting for rollout to finish: 2 out of 3 new replicas have been updated...

  # 或者

  deployment "nginx-deployment" successfully rolled out
  ```

获取关于已更新的 Deployment 的更多信息:

  * 查看 Deployment:
  
  ``` bash
  $ kubectl get deployments
  
  NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
  nginx-deployment   3         3         3            3           36s
  ```
  
  * 查看 Deployment 通过创建新的 ReplicaSet:
  
  ``` bash
  $ kubectl get rs
  
  NAME                          DESIRED   CURRENT   READY   AGE
  nginx-deployment-1564180365   3         3         3       6s
  nginx-deployment-2035384211   0         0         0       36s
  ```

  * 查看 Deployment 的 Pod

  ``` bash
  $ kubectl get pods

  NAME                                READY     STATUS    RESTARTS   AGE
  nginx-deployment-1564180365-khku8   1/1       Running   0          14s
  nginx-deployment-1564180365-nacti   1/1       Running   0          14s
  nginx-deployment-1564180365-z9gth   1/1       Running   0          14s
  ```

  Deployment 可确保在更新时仅关闭一定数量的 Pod。默认情况下，它确保至少所需 Pods 数量的 **75%** 处于运行状态 (最大不可用比例为 25%)。

  Deployment 还确保所创建 Pod 的数量只比期望 Pods 的数量超出一定数值。默认情况下，Deployment 可确保实际启动的 Pod 个数最大为期望值的 125%。

  * 获取 Deployment 的更多信息:

  ``` bash
  kubectl describe deployments
  ```

## 参考文档

* [Kubernetes Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
* [Kubernetes ReplicaSets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
