---
title: Play with Kubernetes
date: 2020-09-04 08:08:52
tags:
  - k8s
---

[Play with Kubernetes](http://labs.play-with-k8s.com/) 是一个可以在浏览器使用的 `CentOS` 虚拟机环境，允许用户通过 `github` 账号登录。在这里，用户可以部署，学习使用 `k8s`。

跳过前面一些很简单的操作。在左侧添加 4 台虚拟机。每一台新创建的虚拟机控制台界面会有如下的提示:

``` text
 1. Initializes cluster master node:

 kubeadm init --apiserver-advertise-address $(hostname -i) --pod-network-cidr 10.5.0.0/16


 2. Initialize cluster networking:

kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter.yaml


 3. (Optional) Create an nginx deployment:

 kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/application/nginx-app.yaml
```

## 搭建 Kubernetes 环境

在 `node-1` 中执行第一条命令，创建 `Master` 节点(虽然有 BLM 运动，但我不在乎)

``` bash
$ kubeadm init --apiserver-advertise-address $(hostname -i) --pod-network-cidr 10.5.0.0/16
...
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.0.38:6443 --token xguam5.v5vgzeifjipno115 \
    --discovery-token-ca-cert-hash sha256:6ea81a284c1b7a5aeec9eb01c8856602f1f3e6f2edd5593816c27224bbccb960
Waiting for api server to startup
Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
daemonset.apps/kube-proxy configured
No resources found
```

然后，从上述命令的输出中找到 `Worker` 节点加入 `Master` 节点的命令，在 `node[2-4]` 节点分别执行命令:

``` bash
kubeadm join 192.168.0.38:6443 --token xguam5.v5vgzeifjipno115 \
    --discovery-token-ca-cert-hash sha256:6ea81a284c1b7a5aeec9eb01c8856602f1f3e6f2edd5593816c27224bbccb960
```

执行完成之后，`node[2-4]` 就成功加入了 `Kubernetes` 集群中。在 `Master` 节点，即 `node-1` 上执行命令验证:

``` bash
$ kubectl get nodes
NAME    STATUS     ROLES    AGE   VERSION
node1   Ready      master   14m   v1.18.4
node2   Ready      <none>   12m   v1.18.4
node3   NotReady   <none>   13s   v1.18.4
node4   NotReady   <none>   5s    v1.18.4
```

最后部署网络插件 `kube-router`:

``` bash
$ kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter.yaml
configmap/kube-router-cfg unchanged
daemonset.apps/kube-router configured
serviceaccount/kube-router unchanged
clusterrole.rbac.authorization.k8s.io/kube-router unchanged
clusterrolebinding.rbac.authorization.k8s.io/kube-router unchanged
```

## 总结

部署完了，用去吧。
