---
title: 投射数据卷 (TBC)
date: 2020-09-04 08:39:17
tags:
  - k8s
  - pod
---

**投射数据卷** 的官方名称为: `projected volume`。`Project` 在这里的意思为 [**投射**](https://cn.bing.com/dict/search?q=project&qs=n&form=Z9LH5&sp=-1&pq=project&sc=8-7&sk=&cvid=A2B01F96E5A847E7A67FEC9AE0A97060)。感谢 `k8s` 帮助我学习英语。

[官方定义为](https://kubernetes.io/docs/concepts/storage/volumes/#projected):

> A projected volume maps several existing volume sources into the same directory.

`Projected Volumes` 存在的意义不是为了存放容器里的数据，也不是用来进行容器和宿主机之间的数据交换。而是为容器提供预先定义好的数据。所以，从容器的角度来看，这些 `Volume` 里的信息就是仿佛是被 `Kubernetes` **投射** (Project) 进入容器当中的。到目前为止，Kubernetes 支持的 Projected Volume 一共有四种:

* Secret
* ConfigMap
* Downward API
* ServiceAccountToken。

下面是官方给出的一个[例子](https://kubernetes.io/docs/concepts/storage/volumes/#example-pod-with-a-secret-a-downward-api-and-a-configmap):

``` yml
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
spec:
  containers:
  - name: container-test
    image: busybox
    volumeMounts:
    - name: all-in-one
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: mysecret
          items:
            - key: username
              path: my-group/my-username
      - downwardAPI:
          items:
            - path: "labels"
              fieldRef:
                fieldPath: metadata.labels
            - path: "cpu_limit"
              resourceFieldRef:
                containerName: container-test
                resource: limits.cpu
      - configMap:
          name: myconfigmap
          items:
            - key: config
              path: my-group/my-config
```

首先，可以看到 `volumes` 与 `containers` 是同级的属性字段，同属于 `pod` 的信息。其次，在上述示例中，使用了三种 `Projected Volume`。

在有了初步认识之后，接下来对每一种 `Projected Volume` 做出更详细的说明。

## Secret

`Secret` 最常见的用法是保存认证信息，比如数据库等。这些数据会被保存在内部的 `ETCD` 中，可以通过将 `Secret` 以 `Volume` 的形式挂载到 `Pod` 上的方式，允许 `Pod` 使用 `Secret` 中的数据。

接下来介绍两种创建 `Secret` 以及使用的方式。

### 命令行方式

#### 通过命令行创建 Secret

使用如下命令创建两个 `Secret`:

``` bash
echo 'admin' > ./user.txt
echo 'password' > ./pass.txt

kubectl create secret generic user --from-file=./user.txt
kubectl create secret generic pass --from-file=./pass.txt
```

#### 查询 Secret

``` bash
$ kubectl get secrets
NAME                  TYPE                                  DATA   AGE
pass                  Opaque                                1      13s
user                  Opaque                                1      18s
```

#### 在 Pod 中使用 Secret

生成一个 `yaml` 文件，引用上面的 `Secret`:

``` bash
# 生成 yaml，在 volumes.projected 中指定上面的 user 与 pass
$ cat << EOF >> busybox.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-projected-volume
spec:
  containers:
  - name: test-secret-volume
    image: busybox
    args:
    - sleep
    - "86400"
    volumeMounts:
    - name: mysql-cred
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: mysql-cred
    projected:
      sources:
      - secret:
          name: user
      - secret:
          name: pass
EOF

# 创建 pod
$ kubectl apply -f ./busybox.yaml
pod/test-projected-volume created
```

#### 在 Pod 中查看 Secret 数据

进入到 `Pod` 中:

``` bash
$ kubectl exec -ti test-projected-volume -- sh
/ #
```

查看 `Secret` 数据:

``` bash
/ # ls /projected-volume/
user.txt
pass.txt
/ # cat /projected-volume/user.txt
admin
/ # cat /projected-volume/pass.txt
password
```

可以看到，在 `mountPath` 指定的路径下面有预先定义好的 `Secret`，并且文件名就是 `--from-file` 指定的参数。

### yaml 方式

#### 通过 yaml 创建 Secret

创建 `Secret` 的配置文件:

``` bash
$ cat << EOF >> secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  user: YWRtaW4=
  pass: cGFzc3dvcmQ=
EOF
```

在 `Kubernetes` 中创建 `Secret`:

``` bash
$ kubectl apply -f ./secret.yml
secret/mysecret created
```

#### 查询 Secret

``` bash
$ kubectl get secrets
NAME                  TYPE                                  DATA   AGE
mysecret              Opaque                                2      3m23s
```

结果与使用命令行的方式有一些区别。

#### 在 Pod 中使用 Secret

生成一个 `yaml` 文件，引用上面的 `Secret`，需要注意 `secret.name` 应使用 `mysecret`，同时，数量从刚才的 **2个** 变成了 **1个**:

``` bash
$ cat << EOF >> busybox.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-projected-volume
spec:
  containers:
  - name: test-secret-volume
    image: busybox
    args:
    - sleep
    - "86400"
    volumeMounts:
    - name: mysql-cred
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: mysql-cred
    projected:
      sources:
      - secret:
          name: mysecret
EOF
```

``` bash
# 创建 pod
$ kubectl apply -f ./busybox.yaml
pod/test-projected-volume created
```

#### 在 Pod 中查看 Secret 数据

进入到 `Pod` 中:

``` bash
$ kubectl exec -ti test-projected-volume -- sh
/ #
```

查看 `Secret` 数据:

``` bash
/ # ls /projected-volume/
pass  user
/ # cat /projected-volume/user
admin
/ # cat /projected-volume/pass.txt
password
```

可以看到，在 `mountPath` 指定的路径下面有预先定义好的 `Secret`，并且文件名就是创建 `Secret` 的 `yaml` 文件中，`data` 字段的 `key`，内容为对应 `value` 经过 `base64` 解码之后的结果。

## Downward API

> A downwardAPI volume is used to make downward API data available to applications. It mounts a directory and writes the requested data in plain text files.

让 `Pod` 里的容器能够直接获取到这个 `Pod API` 对象本身的信息。

`Kubernets` 提供了两种方式使用 `Downward API`:

* Environment variables
* Volume File

### 使用 Pod 的字段作为环境变量

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-envars-fieldref
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "sh", "-c"]
      args:
      - while true; do
          echo -en '\n';
          printenv MY_NODE_NAME MY_POD_NAME MY_POD_NAMESPACE;
          printenv MY_POD_IP MY_POD_SERVICE_ACCOUNT;
          sleep 10;
        done;
      env:
        - name: MY_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: MY_POD_SERVICE_ACCOUNT
          valueFrom:
            fieldRef:
              fieldPath: spec.serviceAccountName
  restartPolicy: Never
```

上述配置文件中，包含了五个环境变量。字段 `env` 是 [EnvVars](https://v1-16.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#envvar-v1-core) 类型的数组。

**fieldRef** 表示选择 **Pod** 的字段，允许使用:

* metadata.name
* metadata.namespace
* metadata.labels
* metadata.annotations
* spec.nodeName
* spec.serviceAccountName
* status.hostIP
* status.podIP

由 `command` 字段与 `args` 字段可以得知，这个 `Pod` 的功能为打印在 `env` 中定义的环境变量。

创建 `Pod`:

``` bash
kubectl apply -f https://k8s.io/examples/pods/inject/dapi-envars-pod.yaml
```

查看 `Pod` 是否为运行状态:

``` bash
kubectl get pods
```

查看日志:

``` bash
kubectl logs dapi-envars-fieldref
```

得到如下输出:

``` bash
minikube
dapi-envars-fieldref
default
172.17.0.4
default
```

### 使用 Container 的字段作为环境变量

与上面的示例类似，除了可以将 `Pod` 的字段作为环境变量的值传入 `Container` 外，同样可以将 `Container` 的字段作为环境变量的值传入 `Container` 中。

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-envars-resourcefieldref
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox:1.24
      command: [ "sh", "-c"]
      args:
      - while true; do
          echo -en '\n';
          printenv MY_CPU_REQUEST MY_CPU_LIMIT;
          printenv MY_MEM_REQUEST MY_MEM_LIMIT;
          sleep 10;
        done;
      resources:
        requests:
          memory: "32Mi"
          cpu: "125m"
        limits:
          memory: "64Mi"
          cpu: "250m"
      env:
        - name: MY_CPU_REQUEST
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              resource: requests.cpu
        - name: MY_CPU_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              resource: limits.cpu
        - name: MY_MEM_REQUEST
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              resource: requests.memory
        - name: MY_MEM_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              resource: limits.memory
  restartPolicy: Never
```

**resourceFieldRef** 表示选择 **Container** 的字段，允许使用 **resources.limits** 和 **resources.requests**，具体如下:

* limits.cpu
* limits.memory
* limits.ephemeral-storage
* requests.cpu
* requests.memory
* requests.ephemeral-storage

创建 `Pod`:

``` bash
kubectl apply -f https://k8s.io/examples/pods/inject/dapi-envars-container.yaml
```

查看 `Pod` 是否为运行状态:

``` bash
kubectl get pods
```

查看日志:

``` bash
kubectl logs dapi-envars-resourcefieldref
```

得到如下输出:

``` bash
1
1
33554432
67108864
```

### 将 Pod 字段存储在文件中

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubernetes-downwardapi-volume-example
  labels:
    zone: us-est-coast
    cluster: test-cluster1
    rack: rack-22
  annotations:
    build: two
    builder: john-doe
spec:
  containers:
    - name: client-container
      image: k8s.gcr.io/busybox
      command: ["sh", "-c"]
      args:
      - while true; do
          if [[ -e /etc/podinfo/labels ]]; then
            echo -en '\n\n'; cat /etc/podinfo/labels; fi;
          if [[ -e /etc/podinfo/annotations ]]; then
            echo -en '\n\n'; cat /etc/podinfo/annotations; fi;
          sleep 5;
        done;
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
  volumes:
    - name: podinfo
      downwardAPI:
        items:
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
          - path: "annotations"
            fieldRef:
              fieldPath: metadata.annotations
```

在这个示例中，为 `Pod` 定义了 `downward API` 类型的 `Volume`，并且将这个 `Volume` 的挂载点位于 `/etc/podinfo`。

`downwardAPI` 的 `item` 属性为 [DownwardAPIVolumeFile](https://v1-16.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#downwardapivolumefile-v1-core) 类型的数组。具体属性为:

|Field|Type|Description|
|:----|:---|:----------|
|fieldRef|[ObjectFieldSelector](https://v1-16.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#objectfieldselector-v1-core)|必须: 从 annotations, labels, name and namespace 之中选择 `Pod` 的一个字段|
|mode|integer|可选:文件属性，取值范围 [0, 0777]。缺省使用默认值。可能与其他影响改文件的参数产生冲突|
|path|string|必须: 文件相对路径，要求必须是相对路径，且不能包含 **..**|
|resourceFieldRef|[ResourceFieldSelector](https://v1-16.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#resourcefieldselector-v1-core)|只支持 `Container` 的 `resources.limit` 与 `resources.requests`|

创建 `Pod`:

``` bash
kubectl apply -f https://k8s.io/examples/pods/inject/dapi-volume.yaml
```

查看 `Pod` 是否为运行状态:

``` bash
kubectl get pods
```

查看日志:

``` bash
kubectl logs kubernetes-downwardapi-volume-example
```

得到如下输出:

``` bash
cluster="test-cluster1"
rack="rack-22"
zone="us-est-coast"

build="two"
builder="john-doe"
```

进入到 `Pod` 中:

``` bash
kubectl exec -it kubernetes-downwardapi-volume-example -- sh
```

查看 `Volume` 文件:

``` bash
/# cat /etc/podinfo/labels
```

得到如下内容:

``` bash
cluster="test-cluster1"
rack="rack-22"
zone="us-est-coast"
```

类似地，查看 `annotations`:

``` bash
/# cat /etc/podinfo/annotations
```

### 将 Container 字段存储在文件中

同样可以将 `Container` 的字段写入到文件中，并以 `Volume` 的形式挂载到 `Container` 中

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubernetes-downwardapi-volume-example-2
spec:
  containers:
    - name: client-container
      image: k8s.gcr.io/busybox:1.24
      command: ["sh", "-c"]
      args:
      - while true; do
          echo -en '\n';
          if [[ -e /etc/podinfo/cpu_limit ]]; then
            echo -en '\n'; cat /etc/podinfo/cpu_limit; fi;
          if [[ -e /etc/podinfo/cpu_request ]]; then
            echo -en '\n'; cat /etc/podinfo/cpu_request; fi;
          if [[ -e /etc/podinfo/mem_limit ]]; then
            echo -en '\n'; cat /etc/podinfo/mem_limit; fi;
          if [[ -e /etc/podinfo/mem_request ]]; then
            echo -en '\n'; cat /etc/podinfo/mem_request; fi;
          sleep 5;
        done;
      resources:
        requests:
          memory: "32Mi"
          cpu: "125m"
        limits:
          memory: "64Mi"
          cpu: "250m"
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
  volumes:
    - name: podinfo
      downwardAPI:
        items:
          - path: "cpu_limit"
            resourceFieldRef:
              containerName: client-container
              resource: limits.cpu
              divisor: 1m
          - path: "cpu_request"
            resourceFieldRef:
              containerName: client-container
              resource: requests.cpu
              divisor: 1m
          - path: "mem_limit"
            resourceFieldRef:
              containerName: client-container
              resource: limits.memory
              divisor: 1Mi
          - path: "mem_request"
            resourceFieldRef:
              containerName: client-container
              resource: requests.memory
              divisor: 1Mi
```

创建 `Pod`:

``` bash
kubectl apply -f https://k8s.io/examples/pods/inject/dapi-volume-resources.yaml
```

进入到 `Pod` 中:

``` bash
kubectl exec -it kubernetes-downwardapi-volume-example-2 -- sh
```

查看 `Volume` 文件:

``` bash
/# cat /etc/podinfo/cpu_limit
```

## 参考文档

* [Projected Volumes](https://kubernetes.io/docs/concepts/storage/volumes/#projected)
* [Configure a Pod to Use a Projected Volume for Storage](https://kubernetes.io/docs/tasks/configure-pod-container/configure-projected-volume-storage/)
* [Donwload API](https://v1-16.docs.kubernetes.io/docs/concepts/storage/volumes/#downwardapi)
* [Expose Pod Information to Containers Through Files](https://v1-16.docs.kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/)
* [Use Pod fields as values for environment variables](https://v1-16.docs.kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/#use-pod-fields-as-values-for-environment-variables)
* [Store Pod fields](https://v1-16.docs.kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/#store-pod-fields)
* [Kubernetes API: EnvVar v1 core](https://v1-16.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#envvar-v1-core)
* [Kubernetes API: EnvVarSource v1 core](https://v1-16.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#envvarsource-v1-core)
