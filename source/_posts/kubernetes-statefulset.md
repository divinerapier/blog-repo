---
title: Kubernetes StatefulSet
date: 2020-10-27 11:58:31
tags:
  - kubernetes
  - controllers
---

StatefulSet 是用来管理有状态应用的工作负载 API 对象。

StatefulSet 用来管理 Deployment 和扩展一组 Pod，并且能为这些 Pod 提供序号和唯一性保证。

和 Deployment 相同的是，StatefulSet 管理了基于相同容器定义的一组 Pod。但和 Deployment 不同的是，StatefulSet 为它们的每个 Pod 维护了一个固定的 ID。这些 Pod 是基于相同的声明来创建的，但是不能相互替换：无论怎么调度，每个 Pod 都有一个永久不变的 ID。

StatefulSet 和其他控制器使用相同的工作模式。你在 StatefulSet 对象 中定义你期望的状态，然后 StatefulSet 的 控制器 就会通过各种更新来达到那种你想要的状态。

## 使用 StatefulSets

StatefulSets 对于需要满足以下一个或多个需求的应用程序很有价值:

* 稳定的、唯一的网络标识符。
* 稳定的、持久的存储。
* 有序的、优雅的部署和缩放。
* 有序的、自动的滚动更新。

在上面，稳定意味着 Pod 调度或重调度的整个过程是有持久性的。如果应用程序不需要任何稳定的标识符或有序的部署、删除或伸缩，则应该使用由一组无状态的副本控制器提供的工作负载来部署应用程序，比如 Deployment 或者 ReplicaSet 可能更适用于您的无状态应用部署需要。

## 限制

* 给定 Pod 的存储必须由 [PersistentVolume](https://github.com/kubernetes/examples/tree/master/staging/persistent-volume-provisioning/README.md) 驱动 基于所请求的 **storage class** 来提供，或者由管理员预先提供。
* 删除或者收缩 StatefulSet 并 **不会删除** 它关联的存储卷。这样做是为了保证数据安全，它通常比自动清除 StatefulSet 所有相关的资源更有价值。
* StatefulSet 当前需要 [Headless Services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) 来负责 Pod 的网络标识。用户需要负责创建此服务。
* 当删除 StatefulSets 时，StatefulSet 不提供任何终止 Pod 的保证。为了实现 StatefulSet 中的 Pod 可以有序和优雅的终止，可以在删除之前将 StatefulSet 缩放为 0。
* 在默认 [Pod 管理策略](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#pod-management-policies)(**OrderedReady**) 时使用 [滚动更新](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#rolling-updates)，可能进入需要 [人工干预](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#forced-rollback) 才能修复的损坏状态。

## 组件

下面的示例演示了 StatefulSet 的组件。

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # by default is 1
  template:
    metadata:
      labels:
        app: nginx # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
```

* 名为 **nginx** 的 Headless Service 用来控制网络域名。
* 名为 **web** 的 StatefulSet 有一个 Spec，它表明将在独立的 3 个 Pod 副本中启动 nginx 容器。
* **volumeClaimTemplates** 将通过 PersistentVolumes 驱动提供的 [PersistentVolumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) 来提供稳定的存储。

StatefulSet 对象的 **name** 必须是合法的 [DNS 域名](https://kubernetes.io/docs/concepts/overview/working-with-objects/names#dns-subdomain-names)。

## Pod Selector

必须将 StatefullSet 的 **.spec.selector** 字段与 **.spec.template.metadata.labels** 设置相同的值。

在 Kubernetes 1.8 版本之前，忽略 **.spec.selector** 字段会获得默认设置值。在 1.8 及以后的版本中，未指定匹配的 Pod Selector 将在创建 StatefulSet 期间导致验证错误。

## 参考文档

* [Kubernetes StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
