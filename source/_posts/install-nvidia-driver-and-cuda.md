---
title: 安装 NVIDIA Driver 和 CUDA
date: 2020-12-08 10:46:02
tags:
  - nvidia
  - cuda
---

## 系统环境

``` bash
$ /home/sihao$ cat /etc/os-release

NAME="Ubuntu"
VERSION="18.04.4 LTS (Bionic Beaver)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 18.04.4 LTS"
VERSION_ID="18.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=bionic
UBUNTU_CODENAME=bionic
```

**GPU: GTX 2080Ti**。

## 下载安装程序

**cuda** 版本为 [cuda10.0](https://developer.nvidia.com/cuda-10.0-download-archive?target_os=Linux&target_arch=x86_64&target_distro=Ubuntu&target_version=1804&target_type=runfilelocal)

**NVIDIA Driver** 版本为 440.82

## 安装 CUDA

``` bash
sudo sh cuda_10.0.130_410.48_linux.run
```

### 配置环境

``` bash
sudo ln -s /usr/local/cuda-10.0 /usr/local/cuda
export PATH="/usr/local/cuda/bin:$PATH"
export LD_LIBRARY_PATH="/usr/local/cuda/lib64:/usr/local/cuda/extras/CUPTI/lib64:$LD_LIBRARY_PATH"
sudo echo "/usr/local/cuda/lib64" >> /etc/ld.so.conf
sudo ldconfig
```

## 安装 NVIDIA Driver

### 卸载已有 NVIDIA Driver

``` bash
sudo /usr/bin/nvidia-uninstall
sudo reboot
```

### 安装新 NVIDIA Driver

``` bash
sudo sh NVIDIA-Linux-x86_64-440.82.run
```

### 验证

``` bash
nvidia-smi
```

## 参考文档

[libcublas-so-10-0-cannot-be-found](https://forums.developer.nvidia.com/t/libcublas-so-10-0-cannot-be-found/69629)
