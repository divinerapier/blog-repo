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

## 参考文档

* [Projected Volumes](https://kubernetes.io/docs/concepts/storage/volumes/#projected)
* [Configure a Pod to Use a Projected Volume for Storage](https://kubernetes.io/docs/tasks/configure-pod-container/configure-projected-volume-storage/)
