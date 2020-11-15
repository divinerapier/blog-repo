---
title: 调度与驱逐 —— 将 Pod 分配到节点上
date: 2020-11-15 18:47:40
tags:
  - kubernetes
  - scheduling
---

Kubernetes 允许用户强制 Pod 只能在特定的 Node(s) 上，或者建议优先在特定的 Node(s) 上运行。常规方法是使用 [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) 来选择。该约束为可选项，默认情况下调度器将自动进行合理的调度，比如，将 pod 分散到节点上，而非在可用资源不足的节点上。但在某些情况下，用户期望对调度 Pod 的 Node(s) 有更多控制，例如，确保 pod 最终落在有 SSD 的机器上，或者将若干有大量通信的服务的 pod 放置在同一个可用区。

## nodeSelector

**nodeSelector** 是最简单推荐形式的节点选择约束。nodeSelector 是 PodSpec 的一个字段，其包含键值对映射。为了使 pod 可以在某个节点上运行，约束键值对构成的集合必须是节点标签集合的子集。

### Get nodes

``` bash
kubectl get nodes
```

### Get the names of cluster's nodes

``` bash
kubectl get nodes
```

### Attach label to the node

规则为:

``` bash
kubectl label nodes <node-name> <label-key>=<label-value>
```

例如，节点 'kubernetes-foo-node-1.c.a-robinson.internal'，标签 'disktype=ssd'，则可以执行:

``` bash
kubectl label nodes kubernetes-foo-node-1.c.a-robinson.internal disktype=ssd
```

通过命令验证:

``` bash
kubectl get nodes --show-labels
```

或者:

``` bash
kubectl describe node <node-name>
```

### Add a nodeSelector field to your pod configuration

如下为原始 Pod 配置文件:

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
```

在此基础上，添加 nodeSelector:

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```

而改文件可以在 https://k8s.io/examples/pods/pod-nginx.yaml 得到。因此，使用如下命令创建 Pod:

``` bash
kubectl apply -f https://k8s.io/examples/pods/pod-nginx.yaml
```

之后，查看 Pod 所在的 Node 并验证约束是否有效:

``` bash
kubectl get pods -o wide

kubectl describe node <node-name>
```

### Affinity and anti-affinity

Affinity and anti-affinity (亲和与反亲和) 是 nodeSelector 提供的一种非常简单的将 pod 约束到具有特定标签的节点上的方法，极大地扩展了用户可以表达约束的类型。关键增强表现为:

1. 语言更具表现力，不只是 “完全匹配的 AND” 语义
1. 规则可以是一种建议性的，而非硬性要求，即使调度器无法找到满足要求的 Node，依旧会调度该 pod
1. 除了可以使用 Node 本身的标签作为约束之外，还可以使用运行在 Node 上的 pod 的标签作为约束，表明可以或者不可以与哪些 pod 运行在同一 Node 上。

Affinity 功能包含两种类型的 affinity: **node affinity** 与 **inter-pod affinity/anti-affinity**。**node affinity** 类似于 **nodeSelector**，对应上述 **1,2** 两点优势。而 **inter-pod affinity/anti-affinity** 具有上述 **1,2,3** 三点优势。

#### Node affinity

**Node affinity** 概念上类似于 **nodeSelector**，可以根据节点上的标签来约束 pod 可以调度到哪些节点。

目前有两种类型的 Node affinity，分别为 **requiredDuringSchedulingIgnoredDuringExecution** 和 **preferredDuringSchedulingIgnoredDuringExecution**。

**requiredDuringSchedulingIgnoredDuringExecution** 指定了将 pod 调度到一个节点上必须满足的规则，原则上等同于 nodeSelector，但语法更具有表现力。

**preferredDuringSchedulingIgnoredDuringExecution** 指定调度器将尝试执行但不能保证的偏好。

名称中 **IgnoredDuringExecution** 类似于 **nodeSelector** 的用法，表明如果节点的标签在 Pod 运行时发生变更，从而不再满足 pod 上的 affinity 规则时，pod 将仍然继续运行在原节点上。**requiredDuringSchedulingRequiredDuringExecution** 还只存在于计划中。

因此，在下面的示例中:

* **requiredDuringSchedulingIgnoredDuringExecution** 的含义为: **必须将 pod 运行在具有 kubernetes.io/e2e-az-name=e2e-az1 或 kubernetes.io/e2e-az-name=e2e-az2 标签的 Node 上**
* **preferredDuringSchedulingIgnoredDuringExecution** 的含义为: **尝试将 pod 运行具有 another-node-label-key=another-node-label-value 标签的 Node 上，如果这不可能的话，则允许 pod 在其他地方运行**

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
```

Node affinity 语法支持的操作符: In，NotIn，Exists，DoesNotExist，Gt，Lt。使用 NotIn 和 DoesNotExist 来实现 **node anti-affinity** 行为，或者使用 [node taints(节点污点将)](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) pod 从特定节点中驱逐。

如果同时指定了 **nodeSelector** 和 **nodeAffinity**，则要求两者必须同时满足，才能将 pod 调度到候选 Node 上。

如果指定了多个与 **nodeAffinity** 类型关联的 **nodeSelectorTerms**，则 Node 只需要满足其中任何一个 nodeSelectorTerms 即可将 pod 调度到 Node 上。

如果指定了多个与 **nodeSelectorTerms** 关联的 **matchExpressions**，则当且仅当所有 **matchExpressions** 得到满足时才将 pod 调度到该 Node 上。

## 参考文档

* [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
