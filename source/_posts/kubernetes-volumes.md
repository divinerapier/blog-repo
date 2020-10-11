---
title: Kubernetes 存储卷
date: 2020-10-07 17:37:14
tags:
  - kubernetes
  - storage
---

## 背景

在 **Docker** 中，**[Volume](https://docs.docker.com/storage/)** 概念表示磁盘上或者另外一个容器内的一个目录。 直到最近，Docker 才支持对基于本地磁盘的 Volume 的生存期进行管理。 虽然 Docker 现在也能提供 Volume 驱动程序，但是目前功能还非常有限 （例如，截至 Docker 1.7，每个容器只允许有一个 Volume 驱动程序，并且无法将参数传递给 Volume）。

而在 **Kubernetes** 中，**Volume** 具有明确的生命周期——与其所属 **Pod** 相同。 因此，**Volume 比 Pod 中运行的任何容器的存活期都长**，在容器重新启动时数据也会得到保留。 当然，**当一个 Pod 不再存在时，卷也将不再存在**。 更重要的是，**Kubernetes 可以支持许多类型的卷，Pod 也能同时使用任意数量的卷**。

**Volume** 的核心是包含一些数据的目录，Pod 中的容器可以访问该目录。 特定的卷类型可以决定这个目录是如何形成的，并能决定它支持何种介质，以及目录中存放什么内容。

使用 **Volume** 时, Pod 声明中需要提供卷的类型 (**.spec.volumes** 字段) 和 **Volume** 挂载的位置 (**.spec.containers.volumeMounts** 字段).

容器中的进程能看到由它们的 Docker 镜像和卷组成的文件系统视图。 [Docker 镜像](https://docs.docker.com/userguide/dockerimages/) 位于文件系统层次结构的根部，并且任何 Volume 都挂载在镜像内的指定路径上。 卷不能挂载到其他卷，也不能与其他卷有硬链接。 Pod 中的每个容器必须独立地指定每个卷的挂载位置(**Volumes** 之间的挂载点应该相互独立)。

## Volume 的类型

Kubernetes 支持下列类型的卷:

* [cephfs](#Cephfs)
* [configMap](#ConfigMap)
* [csi](#CSI)
* [downwardAPI](#DownwardAPI)
* [emptyDir](#EmptyDir)
* [hostPath](#HostPath)
* [local](#Local)
* [nfs](#Nfs)
* [persistentVolumeClaim](#PersistentVolumeClaim)
* [projected](#Projected)
* [secret](#Secret)

### Cephfs

**cephfs** 允许用户将现存的 **CephFS** 卷挂载到 **Pod** 中。 与 **[emptyDir](#EmptyDir)** 不同的是，**emptyDir** 会在删除 Pod 的同时**一并被删除**，**cephfs** 卷的内容在删除 Pod 时会被保留，卷只是被卸载掉了。 这意味着 **CephFS 卷可以被预先填充数据，并且这些数据可以在 Pod 之间"传递"**。CephFS 卷可同时被多个写者挂载。

> 注意： 在您使用 Ceph 卷之前，您的 Ceph 服务器必须正常运行并且要使用的 share 被导出（exported）。

更多信息请参考 [CephFS 示例](https://github.com/kubernetes/examples/tree/master/volumes/cephfs/)。

### ConfigMap

**configMap** 资源提供了向 Pod **注入配置数据**的方法。 ConfigMap 对象中存储的数据可以被 configMap 类型的卷引用，然后被应用到 Pod 中运行的容器化应用。

当引用 configMap 对象时，你可以简单的在 Volume 中通过它名称来引用。 还可以自定义 ConfigMap 中特定条目所要使用的路径。 例如，要将名为 log-config 的 ConfigMap 挂载到名为 configmap-pod 的 Pod 中，您可以使用下面的 YAML:

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
    - name: test
      image: busybox
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: log-config
        items:
          - key: log_level
            path: log_level
```

### CSI

[容器存储接口 (CSI)](https://github.com/container-storage-interface/spec/blob/master/spec.md) 为容器编排系统（如 Kubernetes）定义标准接口，以将任意存储系统暴露给它们的容器工作负载。

更多详情请阅读 [CSI 设计方案](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md)。

CSI 的支持在 Kubernetes v1.9 中作为 alpha 特性引入，在 Kubernetes v1.10 中转为 beta 特性，并在 Kubernetes v1.13 正式 GA。

> **说明:** CSI驱动程序可能并非在所有Kubernetes版本中都兼容。 请查看特定CSI驱动程序的文档，以获取每个 Kubernetes 版本所支持的部署步骤以及兼容性列表。

一旦在 **Kubernetes** 集群上部署了 CSI 兼容卷驱动程序，用户就可以使用 **csi** 作为卷类型来关联、挂载 **CSI Driver** 暴露出来的卷。

允许如下三种方式，在 Pod 中使用 **csi** 类型的卷:

* 通过 **PersistentVolumeClaim**
* 通过 **[Generic ephemeral volumes](https://kubernetes.io/docs/concepts/storage/ephemeral-volumes/#generic-ephemeral-volumes)**
* 通过 **[CSI ephemeral volumes](https://kubernetes.io/docs/concepts/storage/ephemeral-volumes/#csi-ephemeral-volumes)**

存储管理员可以使用以下字段来配置 CSI 持久卷(CSI persistent volume):

* **driver**：指定要使用的卷 驱动程序(CSI Driver) 名称的字符串值。 这个值必须与 **CSI Driver** 的 **[GetPluginInfoResponse]((https://github.com/container-storage-interface/spec/blob/master/spec.md#getplugininfo))** 的 **name** 字段相同。 **Kubernetes** 使用所给的值来标识要调用的 **CSI Driver**；**CSI Driver** 也使用该值来**辨识哪些 PV 对象属于该 CSI Driver**。

* **volumeHandle**：唯一标识卷的字符串值。 该值必须与 **CSI Driver** 的 **[CreateVolumeResponse](https://github.com/container-storage-interface/spec/blob/master/spec.md#createvolume)** 的 **volume.id** 字段相同。 在所有对 **CSI Driver** 的调用中，引用该 **Volume** 时都使用此值作为 **volume_id** 参数。

* **readOnly**：一个可选的布尔值，指示通过 **ControllerPublished** 关联该卷时是否设置该卷为只读。 **默认值是 false**。 该值通过 **[ControllerPublishVolumeRequest](https://github.com/container-storage-interface/spec/blob/master/spec.md#controllerpublishvolume)** 中的 **readonly** 字段传递给 **CSI Driver**。

* **fsType**：如果 **PV** 的 **VolumeMode** 为 **Filesystem**，则该字段指定挂载卷时应该使用的文件系统。 倘若 **Volume** 尚未完成格式化，且支持格式化，则该值将被用于格式化。 可以通过 **[ControllerPublishVolumeRequest](https://github.com/container-storage-interface/spec/blob/master/spec.md#controllerpublishvolume)**、**[NodeStageVolumeRequest](https://github.com/container-storage-interface/spec/blob/master/spec.md#nodestagevolume)** 和 **[NodePublishVolumeRequest](https://github.com/container-storage-interface/spec/blob/master/spec.md#nodepublishvolume)** 的 **volume_capability** 字段将该值传递给 **CSI Driver**。

* **volumeAttributes**：一个 **map[string]string** 类型的映射表，用来设置 **Volume** 的静态属性。 该映射表必须与 **CSI Driver** 返回的 **[CreateVolumeResponse](https://github.com/container-storage-interface/spec/blob/master/spec.md#createvolume)** 中的 volume.attributes 字段的映射相对应。 该映射表通过 **[ControllerPublishVolumeRequest](https://github.com/container-storage-interface/spec/blob/master/spec.md#controllerpublishvolume)**、**[NodeStageVolumeRequest](https://github.com/container-storage-interface/spec/blob/master/spec.md#nodestagevolume)**、和 **[NodePublishVolumeRequest](https://github.com/container-storage-interface/spec/blob/master/spec.md#nodepublishvolume)** 中的 **volume_attributes** 字段传递给 **CSI Driver**。
  * **注意**: 在 [spec](https://github.com/container-storage-interface/spec/blob/master/spec.md) 中只看到了 **volume_context**，并没有 **attributes**，根据注释与数据类型来分析，或许是指这个字段？

* **controllerPublishSecretRef**：对包含敏感信息的 secret 对象的引用；该敏感信息会被传递给 **CSI Driver** 来完成 **[ControllerPublishVolume](https://github.com/container-storage-interface/spec/blob/master/spec.md#controllerpublishvolume)** 和 **[ControllerUnpublishVolume](https://github.com/container-storage-interface/spec/blob/master/spec.md#controllerunpublishvolume)** 调用。 该字段为可选字段；为空表示不需要 secret。 如果 secret 对象包含多个 secret，则所有的 secret 都会被传递。

* **nodeStageSecretRef**：对包含敏感信息的 secret 对象的引用，以传递给 **CSI Driver** 来完成 **[NodeStageVolume](https://github.com/container-storage-interface/spec/blob/master/spec.md#nodestagevolume)** 调用。 该字段为可选字段；为空表示不需要 secret。 如果 secret 对象包含多个 secret，则所有的 secret 都会被传递。

* **nodePublishSecretRef**：对包含敏感信息的 secret 对象的引用，以传递给 **CSI Driver** 来完成 **[NodePublishVolume](https://github.com/container-storage-interface/spec/blob/master/spec.md#nodepublishvolume)** 调用。 该字段为可选字段；为空表示不需要 secret。 如果 secret 对象包含多个 secret，则所有的 secret 都会被传递。

### DownwardAPI

**downwardAPI** 类型的 **Volume** 被用于使 **downward API** 数据对应用程序可见。其表现形式为，挂载一个目录，并将请求的数据写入到纯文本文件中。

### EmptyDir

当 **Pod** 被指定到某个节点上时，首先创建的是一个 **emptyDir** 类型的 **Volume**，并且只要 **Pod** 保持在该节点上运行，**Volume** 就一直存在。正如名字所说的那样，**Volume** 的初始状态为空。虽然 **Pod** 中的容器挂载 **emptyDir** 类型 **Volume** 的路径可能不尽相同，但这都不重要，重要的是，这些容器都可以读写 **emptyDir** 类型 **Volume** 中的相同的文件。 无论因何种原因，只要 **Pod** 从节点上被删除，**emptyDir** 类型的 **Volume** 中的数据也会被永久删除。

> **说明**: 容器崩溃并不会导致 **Pod** 从节点上被移除，因此容器崩溃时 **emptyDir** 类型 **Volume** 中的数据是安全的。

有如下需求可以考虑使用 **emptyDir** 类型 **Volume**:

* 缓存空间，例如基于磁盘的归并排序。
* 为耗时较长的计算任务提供检查点，以便任务能方便地从崩溃前状态恢复执行。
* 在 Web 服务器容器服务数据时，保存内容管理器容器获取的文件。

默认情况下， **emptyDir volume** 所使用的的实际存储介质由节点使用何种存储介质决定: 可以是 **HDD** 或 **SSD** 或 **NFS** 等。但是，可以令 **emptyDir.medium = Memory** 使 **Kubernetes** 安装 **tmpfs**。但需要考虑到，**tmpfs** 的优势与劣势都很突出:

* 优势: 基于 **RAM** 的文件系统，速度非常快
* 劣势: 随节点重启被清除，且写入的所有文件都会计入容器的内存消耗，受容器内存限制约束

#### EmptyDir 示例

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

### HostPath

### Local

### Nfs

### PersistentVolumeClaim

### Projected

### Secret

## 参考文档

* [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
