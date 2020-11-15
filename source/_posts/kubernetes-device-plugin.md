---
title: Kubernetes Device Plugin
date: 2020-11-15 17:20:26
tags:
---

Kubernetes 提供 [device plugin framework](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/resource-management/device-plugin.md)，允许用户将系统硬件资源发布到 Kubelet。

Device Plugin 由设备供应商实现，由用户或手动部署或作为 DaemonSet 来部署，而无需定制 Kubernetes 本身的代码。目标设备可以是 GPU、高性能 NIC、FPGA、InfiniBand 适配器以及其他类似的、可能需要特定于供应商的初始化和设置的计算资源。

## 注册 Device Plugin

kubelet 提供了一个 Registration 的 gRPC 服务:

``` protobuf
service Registration {
    rpc Register(RegisterRequest) returns (Empty) {}
}
```

**Device Plugin** 可以通过此 gRPC 服务在 kubelet 进行注册。在注册时，**Device Plugin** 需要提供如下内容:

* Device Plugin 的 Unix 套接字。
* Device Plugin 的 API 版本。
* ResourceName。遵循 [扩展资源命名方案](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#extended-resources)，形如 vendor-domain/resourcetype: 比如 NVIDIA GPU 就被公布为 nvidia.com/gpu。

在成功注册后，**Device Plugin** 会向 kubelet 发送他所管理的设备列表，之后 kubelet 负责将这些资源发布到 API Server，作为 kubelet 节点状态更新的一部分。

比如，**Device Plugin** 在 kubelet 中注册了 **hardware-vendor.example/foo** 并报告了节点上的两个运行状况良好的设备后，节点状态将更新以通告该节点已安装2个 Foo 设备并且是可用的。

然后，用户就可以在 Container 规范中请求这类设备，但是有以下的限制:

* 扩展资源仅可作为整数资源使用，且不能被过量使用
* 设备不能在容器之间共享

假设 Kubernetes 集群正在运行一个 **Device Plugin**，ResourceName 为 **hardware-vendor.example/foo**。下面就是一个 Pod 示例，请求此资源以运行某演示负载：

``` yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
spec:
  containers:
    - name: demo-container-1
      image: k8s.gcr.io/pause:2.0
      resources:
        limits:
          hardware-vendor.example/foo: 2
#
# pod 需要两个 hardware-vendor.example/foo 设备
# 而且只能够调度到满足需求的 node 上
#
# 如果该节点中有2个以上的设备可用，剩余的设备可供其他 pod 使用
```

## 实现 Device Plugin

Device Plugin 的常规工作流程包括以下几个步骤：

* 初始化。在这个阶段，Device Plugin 将执行供应商特定的初始化和设置，以确保设备处于就绪状态。
* 使用主机路径 /var/lib/kubelet/device-plugins/ 下的 Unix socket 启动一个 gRPC 服务，该服务实现以下接口：

    ``` protobuf
    service DevicePlugin {
        // GetDevicePluginOptions returns options to be communicated with Device Manager.
        rpc GetDevicePluginOptions(Empty) returns (DevicePluginOptions) {}

        // ListAndWatch returns a stream of List of Devices
        // Whenever a Device state change or a Device disappears, ListAndWatch
        // returns the new list
        rpc ListAndWatch(Empty) returns (stream ListAndWatchResponse) {}

        // Allocate is called during container creation so that the Device
        // Plugin can run device specific operations and instruct Kubelet
        // of the steps to make the Device available in the container
        rpc Allocate(AllocateRequest) returns (AllocateResponse) {}

        // GetPreferredAllocation returns a preferred set of devices to allocate
        // from a list of available ones. The resulting preferred allocation is not
        // guaranteed to be the allocation ultimately performed by the
        // devicemanager. It is only designed to help the devicemanager make a more
        // informed allocation decision when possible.
        rpc GetPreferredAllocation(PreferredAllocationRequest) returns (PreferredAllocationResponse) {}

        // PreStartContainer is called, if indicated by Device Plugin during registeration phase,
        // before each container start. Device plugin can run device specific operations
        // such as resetting the device before making devices available to the container.
        rpc PreStartContainer(PreStartContainerRequest) returns (PreStartContainerResponse) {}
    }
    ```

* 通过 Unix socket 在主机路径 /var/lib/kubelet/device-plugins/kubelet.sock 处向 kubelet 注册自身。
* 成功注册自身后，Device Plugin 将以服务模式运行，之后，它将持续监控设备运行状况，并在设备状态发生任何变化时报告 kubelet。它还负责响应 Allocate gRPC 请求。 在 Allocate 期间，Device Plugin 可能还会做一些设备特定的准备；例如清理 GPU 或初始化 QRNG。如果操作成功，则 Device Plugin 将返回 AllocateResponse，其中包含用于访问被分配的设备容器运行时的配置。 kubelet 将此信息传递到容器运行时。

### 处理 kubelet 重启

Device Plugin 应能监测到 kubelet 重启，并且向新的 kubelet 实例来重新注册自己。在当前实现中，当 kubelet 重启的时候，新的 kubelet 实例会删除 /var/lib/kubelet/device-plugins 下所有已经存在的 Unix sockets。 Device Plugin 需要能够监控到它的 Unix socket 被删除，并且当发生此类事件时重新注册自己。

## 部署 Device Plugin

用户可以将 Device Plugin 作为节点操作系统的软件包来部署、作为 DaemonSet 来部署或者手动部署。

规范目录 /var/lib/kubelet/device-plugins 是需要特权访问的，所以 Device Plugin 必须要在被授权的安全的上下文中运行。如果将 Device Plugin 部署为 DaemonSet，/var/lib/kubelet/device-plugins 目录必须要在 DevicePlugin 的 PodSpec 中声明作为 Volume 被 mount 到 Device Plugin 中。

若选择 DaemonSet 方法，用户可以通过 Kubernetes 进行以下操作: 将 Device Plugin 的 Pod 放置在节点上，在出现故障后重新启动守护进程 Pod，来进行自动升级。

## API 兼容性

Kubernetes Device Plugin 还处于 beta 版本。所以在稳定版本出来之前 API 会以不兼容的方式进行更改。作为一个项目，Kubernetes 建议 Device Plugin 开发者:

* 注意未来版本的更改
* 支持多个版本的 Device Plugin API，以实现向后/向前兼容性。

如果你启用 DevicePlugins 功能，并在需要升级到 Kubernetes 版本来获得较新的 Device Plugin API 版本的节点上运行 Device Plugin，请在升级这些节点之前先升级 Device Plugin 以支持这两个版本。 采用该方法将确保升级期间设备分配的连续运行。

## 监控 Device Plugin

为了监控 Device Plugin 提供的资源，监控代理程序需要能够发现节点上正在使用的设备，并获取元数据来描述哪个指标与容器相关联。 设备监控代理暴露给 [Prometheus](https://prometheus.io/) 的指标应该遵循 [Kubernetes Instrumentation Guidelines](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-instrumentation/instrumentation.md)，使用 pod、namespace 和 container 标签来标识容器。

kubelet 提供了 gRPC 服务来使得正在使用中的设备被发现，并且还未这些设备提供了元数据:

``` protobuf
// PodResourcesLister is a service provided by the kubelet that provides information about the
// node resources consumed by pods and containers on the node
service PodResourcesLister {
    rpc List(ListPodResourcesRequest) returns (ListPodResourcesResponse) {}
}
```

gRPC 服务通过 /var/lib/kubelet/pod-resources/kubelet.sock 的 UNIX 套接字来提供服务。 Device Plugin资源的监控代理程序可以部署为守护进程或者 DaemonSet。 规范的路径 /var/lib/kubelet/pod-resources 需要特权来进入， 所以监控代理程序必须要在获得授权的安全的上下文中运行。 如果设备监控代理以 DaemonSet 形式运行，必须要在插件的 PodSpec 中声明将 /var/lib/kubelet/pod-resources 目录以 卷的形式被挂载到容器中。

对“PodResources 服务”的支持要求启用 KubeletPodResources 特性门控。 从 Kubernetes 1.15 开始默认启用。

## Device Plugin 集成 The Topology Manager

The Topology Manager 是 Kubelet 的一个组件，它允许以拓扑对齐方式来调度资源。 为了做到这一点，Device Plugin API 进行了扩展来包括一个 TopologyInfo 结构体。

``` protobuf
message TopologyInfo {
    repeated NUMANode nodes = 1;
}

message NUMANode {
    int64 ID = 1;
}
```

Device Plugin 希望 The Topology Manager 可以将填充的 TopologyInfo 结构体作为设备注册的一部分以及设备 ID 和设备的运行状况发送回去。然后 The Topology Manager 将使用此信息来咨询拓扑管理器并做出资源分配决策。

TopologyInfo 支持定义 nodes 字段，允许为 nil（默认）或者是一个 NUMA 节点的列表。 这样就可以使Device Plugin可以跨越 NUMA 节点去发布。

下面是一个由 Device Plugin 为设备填充 TopologyInfo 结构体的示例:

``` text
pluginapi.Device{ID: "25102017", Health: pluginapi.Healthy, Topology:&pluginapi.TopologyInfo{Nodes: []*pluginapi.NUMANode{&pluginapi.NUMANode{ID: 0,},}}}
```

## 参考文档

* [Kubernetes Device Plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
