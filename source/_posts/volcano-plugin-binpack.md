---
title: Volcano Plugin - binpack
date: 2020-12-07 15:17:39
tags:
  - kubernetes
  - volcano
---

## 目标

**binpack** 解决的问题是，提高节点的利用率，避免资源碎片化。

## 源码分析

如下代码来自: `pkg/scheduler/plugins/binpack/binpack.go`。

``` go
// BinPackingScore use the best fit polices during scheduling.
// Goals:
// - Schedule Jobs using BestFit Policy using Resource Bin Packing Priority Function
// - Reduce Fragmentation of scarce resources on the Cluster
func BinPackingScore(task *api.TaskInfo, node *api.NodeInfo, weight priorityWeight) float64 {
    score := 0.0
    weightSum := 0
    requested := task.Resreq
    allocatable := node.Allocatable
    used := node.Used

    for _, resource := range requested.ResourceNames() {
        request := requested.Get(resource)
        if request == 0 {
            continue
        }
        allocate := allocatable.Get(resource)
        nodeUsed := used.Get(resource)

        resourceWeight := 0
        found := false
        switch resource {
        case v1.ResourceCPU:
            resourceWeight = weight.BinPackingCPU
            found = true
        case v1.ResourceMemory:
            resourceWeight = weight.BinPackingMemory
            found = true
        default:
            resourceWeight, found = weight.BinPackingResources[resource]
        }
        if !found {
            continue
        }

        resourceScore := ResourceBinPackingScore(request, allocate, nodeUsed, resourceWeight)
        klog.V(5).Infof("task %s/%s on node %s resource %s, need %f, used %f, allocatable %f, weight %d, score %f", task.Namespace, task.Name, node.Name, resource, request, nodeUsed, allocate, resourceWeight, resourceScore)

        score += resourceScore
        weightSum += resourceWeight
    }

    // mapping the result from [0, weightSum] to [0, 10(MaxPriority)]
    if weightSum > 0 {
        score /= float64(weightSum)
    }
    score *= float64(v1alpha1.MaxNodeScore * int64(weight.BinPackingWeight))

    return score
}

// ResourceBinPackingScore calculate the binpack score for resource with provided info
func ResourceBinPackingScore(requested, capacity, used float64, weight int) float64 {
    if capacity == 0 || weight == 0 {
        return 0
    }

    usedFinally := requested + used
    if usedFinally > capacity {
        return 0
    }

    score := usedFinally * float64(weight) / capacity
    return score
}
```

## 配置文件

``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: volcano-scheduler-configmap
  namespace: volcano-system
data:
  volcano-scheduler.conf: |
    actions: "enqueue, allocate, backfill"
    tiers:
    - plugins:
      - name: priority
      - name: gang
      - name: conformance
    - plugins:
      - name: drf
      - name: predicates
      - name: proportion
      - name: nodeorder
      - name: binpack
        arguments:
          # binpack 插件权重
          - binpack.weight: 10
          # cpu 资源权重
          - binpack.cpu: 1
          # memory 资源权重
          - binpack.memory: 1
          # gpu 等其他资源类型
          - binpack.resources: nvidia.com/gpu
          # gpu 等其他资源权重配置
          - binpack.resources.nvidia.com/gpu: 2
```

### Volcano Scheduler 日志

**Volcano Scheduler** 中可以看到 **binpack plugin** 加载的配置文件:

``` bash
$ kubectl -n volcano-system logs -f volcano-scheduler-566b6f749d-4wr6m

I1208 07:27:43.875761       1 binpack.go:161] Leaving binpack plugin. binpack.weight[10], binpack.cpu[1], binpack.memory[1], nvidia.com/gpu[2] ...
```
