---
title: 使用 kubeadm 创建 kubernetes 集群
date: 2020-10-19 20:38:25
tags:
  - kubeadm
  - kubernetes
---

在 **Ubuntu 20.04** 系统上搭建 **Kubernetes** 集群。

## 安装 Docker

``` bash
sudo apt-get remove docker docker-engine docker.io containerd runc
sudo apt-get update
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update --fix-missing
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl enable docker
```

## 下载 Kubernetes 相关程序

``` bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
sudo apt-get update --fix-missing
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## 配置桥接网络防火墙

``` bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

## 关闭 swap

``` bash
# 临时关闭
swapoff -a

# 永久关闭，注释 swap 配置
vi /etc/fstab
```

## 关闭防火墙

``` bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

## 关闭 selinux

``` bash
sudo sed -i 's/enforcing/disabled/' /etc/selinux/config
sudo setenforce 0

```

## 拉取 gcr 镜像

需要在所有的 **Master** 节点与 **Worker** 节点拉取镜像。

**bash** 环境使用如下脚本:

``` bash
image_list=$(kubeadm config images list)

for image in ${image_list} ; do
  name=$(echo ${image} | cut -d'/' -f2)
  docker pull registry.aliyuncs.com/google_containers/$name
  docker image tag registry.aliyuncs.com/google_containers/$name k8s.gcr.io/$name
  docker image rm registry.aliyuncs.com/google_containers/$name
done
```

**zsh** 环境使用如下脚本:

``` zsh
image_list=$(kubeadm config images list)
images=(`echo ${image_list} | tr '\n' ' '`)

for image in ${images} ; do
  name=$(echo ${image} | cut -d'/' -f2)
  docker pull registry.aliyuncs.com/google_containers/$name
  docker image tag registry.aliyuncs.com/google_containers/$name k8s.gcr.io/$name
  docker image rm registry.aliyuncs.com/google_containers/$name
done
```

## 初始化 Master 节点

在 **Master** 节点执行命令:

``` bash
sudo kubeadm init \
  --apiserver-advertise-address $(hostname -i) \
  --pod-network-cidr 10.244.0.0/16 --v=5

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.50.5:6443 --token 7zjyq9.x3xkoatt6pb1cbsu \
    --discovery-token-ca-cert-hash sha256:1c78c44bc57e6e887c5f81e7a9c6c3e52f098e1ba9255f5303ac78129d410774
```

或者省略下载镜像步骤，直接创建集群:

``` bash
sudo kubeadm init \
  --apiserver-advertise-address=$(hostname -i) \
  --image-repository registry.aliyuncs.com/google_containers \
  --service-cidr=10.5.0.0/16 \
  --pod-network-cidr=10.244.0.0/16
```

然后配置 **kubeconfig**:

``` bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 配置网络

### Calico

``` bash
kubectl apply -f https://docs.projectcalico.org/manifests/canal.yaml
```

## 添加 Worker 节点

在 **Worker** 节点执行命令:

``` bash
sudo kubeadm join 192.168.50.5:6443 --token 4n2pwp.hq9jyo3auaibma3q     --discovery-token-ca-cert-hash sha256:750da2c87a67b96bfec73ade40888d22b61e045fdd28bbb7a4ff2c6ce3e0309c

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

## 测试集群

``` bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/application/nginx-app.yaml
```

## 清理集群

在期望清理的节点执行:

``` bash
sudo kubeadm reset
sudo rm -rf /etc/cni/net.d
rm -rf ~/.kube
```

## 参考文档

* [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
* [Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
* [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
* [Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
* [Deploying kube-router with kubeadm](https://github.com/cloudnativelabs/kube-router/blob/master/docs/kubeadm.md)
* [Install Calico for policy and flannel (aka Canal) for networking](https://docs.projectcalico.org/getting-started/kubernetes/flannel/flannel)
