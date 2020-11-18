---
title: Volcano
date: 2020-11-16 14:43:43
tags:
  - kubernetes
  - scheduler
---

## Installation

从 [volocano release](https://github.com/volcano-sh/volcano/releases) 下载 **Volcano**，通过 **yaml** 文件创建 **Deployment** 等。

以当前版本 **v1.1.0** 为例:

``` bash
$ wget https://github.com/volcano-sh/volcano/releases/download/v1.1.0/volcano-v1.1.0-linux-gnu.tar.gz

$ tar xzf volcano-v1.1.0-linux-gnu.tar.gz

$ kubectl apply -f ./volcano-v1.1.0.yaml
namespace/volcano-system created
namespace/volcano-monitoring created
configmap/volcano-scheduler-configmap created
serviceaccount/volcano-scheduler created
clusterrole.rbac.authorization.k8s.io/volcano-scheduler created
clusterrolebinding.rbac.authorization.k8s.io/volcano-scheduler-role created
deployment.apps/volcano-scheduler created
service/volcano-scheduler-service created
serviceaccount/volcano-admission created
clusterrole.rbac.authorization.k8s.io/volcano-admission created
clusterrolebinding.rbac.authorization.k8s.io/volcano-admission-role created
deployment.apps/volcano-admission created
service/volcano-admission-service created
job.batch/volcano-admission-init created
serviceaccount/volcano-controllers created
clusterrole.rbac.authorization.k8s.io/volcano-controllers created
clusterrolebinding.rbac.authorization.k8s.io/volcano-controllers-role created
deployment.apps/volcano-controllers created
Warning: apiextensions.k8s.io/v1beta1 CustomResourceDefinition is deprecated in v1.16+, unavailable in v1.22+; use apiextensions.k8s.io/v1 CustomResourceDefinition
customresourcedefinition.apiextensions.k8s.io/jobs.batch.volcano.sh created
customresourcedefinition.apiextensions.k8s.io/commands.bus.volcano.sh created
customresourcedefinition.apiextensions.k8s.io/podgroups.scheduling.volcano.sh created
customresourcedefinition.apiextensions.k8s.io/queues.scheduling.volcano.sh created
```

验证 **Volcano** 组件运行状态:

``` bash
$ kubectl get all -n volcano-system
NAME                                     READY   STATUS      RESTARTS   AGE
pod/volcano-admission-7cfdf5b8d-cd2mk    1/1     Running     0          6m27s
pod/volcano-admission-init-rmd7w         0/1     Completed   0          6m27s
pod/volcano-controllers-c4c5f48b-dtx4w   1/1     Running     0          6m27s
pod/volcano-scheduler-54f77d6788-d6t9j   1/1     Running     0          6m27s

NAME                                TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/volcano-admission-service   ClusterIP   10.5.51.59    <none>        443/TCP    6m27s
service/volcano-scheduler-service   ClusterIP   10.5.128.19   <none>        8080/TCP   6m27s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/volcano-admission     1/1     1            1           6m27s
deployment.apps/volcano-controllers   1/1     1            1           6m27s
deployment.apps/volcano-scheduler     1/1     1            1           6m27s

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/volcano-admission-7cfdf5b8d    1         1         1       6m27s
replicaset.apps/volcano-controllers-c4c5f48b   1         1         1       6m27s
replicaset.apps/volcano-scheduler-54f77d6788   1         1         1       6m27s

NAME                               COMPLETIONS   DURATION   AGE
job.batch/volcano-admission-init   1/1           4m24s      6m27s
```

## 创建任务

### CPU 任务

``` yaml
apiVersion: kubeflow.org/v1
kind: MPIJob
metadata:
  name: openmpi-helloworld-job
spec:
  schedulerName: volcano
  slotsPerWorker: 1
  cleanPodPolicy: Running
  mpiReplicaSpecs:
    Launcher:
      replicas: 1
      template:
         spec:
           containers:
           - image: divinerapier/openmpi-helloworld:0.0.1
             name: openmpi-helloworld-job
             command:
             - mpirun
             - --allow-run-as-root
             - -np
             - "2"
             - /helloworld/mpi_hello_world
             resources:
               request:
                 cpu: 0.1
               limits:
                 cpu: 0.1
    Worker:
      replicas: 2
      template:
        spec:
          containers:
          - image: divinerapier/openmpi-helloworld:0.0.1
            name: openmpi-helloworld-job
            resources:
              request:
                cpu: 0.1
              limits:
                cpu: 0.1
```

### GPU 任务

``` yaml
apiVersion: kubeflow.org/v1
kind: MPIJob
metadata:
  name: tensorflow-benchmarks
spec:
  schedulerName: volcano
  slotsPerWorker: 1
  cleanPodPolicy: Running
  mpiReplicaSpecs:
    Launcher:
      replicas: 1
      template:
         spec:
           containers:
           - image: mpioperator/tensorflow-benchmarks:latest
             name: tensorflow-benchmarks
             command:
             - mpirun
             - --allow-run-as-root
             - -np
             - "2"
             - -bind-to
             - none
             - -map-by
             - slot
             - -x
             - NCCL_DEBUG=INFO
             - -x
             - LD_LIBRARY_PATH
             - -x
             - PATH
             - -mca
             - pml
             - ob1
             - -mca
             - btl
             - ^openib
             - python
             - scripts/tf_cnn_benchmarks/tf_cnn_benchmarks.py
             - --model=resnet101
             - --batch_size=64
             - --variable_update=horovod
             resources:
               limits:
                 nvidia.com/gpu: 1
    Worker:
      replicas: 2
      template:
        spec:
          containers:
          - image: mpioperator/tensorflow-benchmarks:latest
            name: tensorflow-benchmarks
            resources:
              limits:
                nvidia.com/gpu: 1
```

## 注意事项

**volcano-scheduler** 在调度任务时，当任务使用的资源太少时会被跳过，具体逻辑为:

``` go
var (
    minMilliCPU float64 = 10
    minMilliScalarResources float64 = 10
    minMemory float64 = 1
)

func (alloc *Action) Execute(ssn *framework.Session) {
    // ...
    if _, found = pendingTasks[job.UID]; !found {
        tasks := util.NewPriorityQueue(ssn.TaskOrderFn)
        for _, task := range job.TaskStatusIndex[api.Pending] {
            // Skip BestEffort task in 'allocate' action.
            if task.Resreq.IsEmpty() {
                klog.V(4).Infof("Task <%v/%v> is BestEffort task, skip it.",
                task.Namespace, task.Name)
                continue
            }

            tasks.Push(task)
        }
        pendingTasks[job.UID] = tasks
    }
    // ...
}

// IsEmpty returns bool after checking any of resource is less than min possible value
func (r *Resource) IsEmpty() bool {
    if r.MilliCPU >= minMilliCPU || r.Memory >= minMemory {
        return false
    }

    for _, rQuant := range r.ScalarResources {
        if rQuant >= minMilliScalarResources {
            return false
        }
    }

    return true
}
```

所以，在使用 **volcano** 作为调度器时，必须要对 **Pod** 使用的资源做出限制。对于使用 **volcano** 调度 **MPIJob** 时，无论是 **Launcher** 还是 **Worker** 都需要显示声明需要的资源。

更具体地，**volcano** 会将资源分为两个大类:

* **CPU** 与 **Memory**
* 其他资源

要求上述两类资源，至少有一类使用的资源满足最低要求即可。
