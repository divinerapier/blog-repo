---
title: Kubernetes DaemonSet
date: 2020-10-28 14:41:29
tags:
  - kubernetes
  - controllers
---

**DaemonSet** 确保全部 (或者某些) 节点上运行一个 Pod 的副本。 当有节点加入集群时，也会为他们新增一个 Pod 。当有节点从集群移除时，这些 Pod 也会被回收。删除 DaemonSet 将会删除它创建的所有 Pod。

DaemonSet 的一些典型用法:

* 在每个节点上运行集群守护进程
  * 比如: 网络插件，存储插件
* 在每个节点上运行日志收集守护进程
* 在每个节点上运行监控守护进程

## 创建 DaemonSet

下面的 **daemonset.yaml** 文件描述了一个运行 **fluentd-elasticsearch** Docker 镜像的 DaemonSet:

``` yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # this toleration is to have the daemonset runnable on master nodes
      # remove it if your masters can't run pods
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

使用 **yaml** 文件创建 **DaemonSet**:

``` bash
kubectl apply -f https://k8s.io/examples/controllers/daemonset.yaml
```

## 如何调度 Daemon Pods

### 通过默认调度器调度

DaemonSet 确保所有符合条件的节点都运行该 Pod 的一个副本。 通常，运行 Pod 的节点由 Kubernetes 调度器选择。不过，DaemonSet pods 由 DaemonSet 控制器创建和调度。这就带来了以下问题:

* Pod 行为的不一致性: 正常 Pod 在被创建后等待调度时处于 Pending 状态， DaemonSet Pods 创建后不会处于 Pending 状态下。这使用户感到困惑。
* [Pod 抢占](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/) 由默认调度器处理。启用抢占后，DaemonSet 控制器将在不考虑 Pod 优先级和抢占 的情况下制定调度决策。

**ScheduleDaemonSetPods** 控制 Kubernetes 使用 **默认调度器** 而不是 **DaemonSet 控制器** 来调度 DaemonSets，通过将 **yaml** 配置文件中 **Pod** 部分的 **.spec.nodeName** 替换为 **.spec.affinity.nodeAffinity**。更多内容请点击 [Assigning Pods to Nodes: Affinity and anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)。

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

* **requiredDuringSchedulingIgnoredDuringExecution**: 必须将 Pod 部署到满足条件的节点上，否则不断重试
* **preferredDuringSchedulingIgnoredDuringExecution**: 优先将 Pod 部署到满足条件的节点上，否则忽略该条件

此外，系统会自动添加 **node.kubernetes.io/unschedulable: NoSchedule** 容忍度到 **DaemonSet Pods**。在调度 DaemonSet Pod 时，默认调度器会忽略 **unschedulable** 节点。

## 参考文档

* [Kubernetes DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
* [Pod Priority and Preemption](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/)
* [Assigning Pods to Nodes: Affinity and anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)
