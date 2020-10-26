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
kubectl create -f deploy/v1alpha2/mpi-operator.yaml
```

验证是否安装成功:

``` bash
kubectl get crd
# NAME                                          CREATED AT
# mpijobs.kubeflow.org                          2020-10-23T08:40:15Z
```

## 使用

创建一个 **MPIJob** 的配置文件:

``` yaml
apiVersion: kubeflow.org/v1alpha2
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
               limits:
                 cpu: 10Mi
                 memory: 10Mi
    Worker:
      replicas: 2
      template:
        spec:
          containers:
          - image: divinerapier/openmpi-helloworld:0.0.1
            name: openmpi-helloworld
            resources:
              limits:
                cpu: 10Mi
                memory: 10Mi
```

部署到 **Kubernetes** 上:

``` bash
kubectl apply -f ./openmpi-helloworld.yml
```

## 参考文档

* [GitHub: MPI Operator](https://github.com/kubeflow/mpi-operator)
* [Introduction to Kubeflow MPI Operator and Industry Adoption](https://medium.com/kubeflow/introduction-to-kubeflow-mpi-operator-and-industry-adoption-296d5f2e6edc)
* [MPI Training](https://www.kubeflow.org/docs/components/training/mpi/)
