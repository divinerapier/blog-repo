---
title: MPI Operator
date: 2020-10-24 18:17:12
tags:
  - kubernetes
  - mpi
  - operator
---

## 安装

部署默认配置的 **mpi-operator**:

``` bash
git clone https://github.com/kubeflow/mpi-operator
cd mpi-operator
kubectl create -f deploy/v1/mpi-operator.yaml
```

验证是否安装成功:

``` bash
kubectl get crd
# NAME                                          CREATED AT
# mpijobs.kubeflow.org                          2020-10-23T08:40:15Z
```

### 参数选项

在使用 **v1** 版本时，需要注意几个选项:

* **-namespace**: 不为空时，只监控指定 **namespace** 的 **MPIJob**，否则将监控所有的 **namespace**
* **-gang-scheduling**: 指定使用的 **gang scheduler** 的名字，此时会启动 **gang scheduling** 调度策略
* **launcher-runs-workloads**: 在 **launcher** 拥有 **GPU** 时执行任务

## 使用

创建一个 **MPIJob** 的配置文件:

``` yaml
apiVersion: kubeflow.org/v1
kind: MPIJob
metadata:
  name: openmpi-helloworld
spec:
  slotsPerWorker: 1
  cleanPodPolicy: Running
  mpiReplicaSpecs:
    Launcher:
      replicas: 1
      template:
         spec:
           containers:
           - image: divinerapier/openmpi-helloworld:0.0.1
             name: openmpi-helloworld
             command:
             - mpirun
             - --allow-run-as-root
             - -np
             - "2"
             - /helloworld/mpi_hello_world
             resources:
               request:
                cpu: 0.1
                memory: 1Gi
               limits:
                 cpu: 0.1
                 memory: 1Gi
    Worker:
      replicas: 2
      template:
        spec:
          containers:
          - image: divinerapier/openmpi-helloworld:0.0.1
            name: openmpi-helloworld
            resources:
              request:
                cpu: 0.1
                memory: 1Gi
              limits:
                cpu: 0.1
                memory: 1Gi
```

部署到 **Kubernetes** 上:

``` bash
kubectl apply -f ./openmpi-helloworld.yml
```

## 参考文档

* [GitHub: MPI Operator](https://github.com/kubeflow/mpi-operator)
* [Introduction to Kubeflow MPI Operator and Industry Adoption](https://medium.com/kubeflow/introduction-to-kubeflow-mpi-operator-and-industry-adoption-296d5f2e6edc)
* [MPI Training](https://www.kubeflow.org/docs/components/training/mpi/)
