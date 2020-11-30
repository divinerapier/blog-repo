---
title: 设置容器内的 locale
date: 2020-11-30 10:22:26
tags:
  - container
  - linux
---

解决办法面向 **Ubuntu/Debian** 系列，**CentOS** 系列方法类似。

## 在容器内处理

``` bash
apt update --fix-missing
apt install -y locales
sed -i '/en_US.UTF-8/s/^# //g' /etc/locale.gen
locale-gen
echo "export LANG=en_US.UTF-8" >> ~/.bashrc
echo "export LANGUAGE=en_US.UTF-8" >> ~/.bashrc
echo "export LC_ALL=en_US.UTF-8" >> ~/.bashrc
```

## 在 Dockerfile 中处理

``` dockerfile
RUN apt update --fix-missing \
    && apt install -y locales \
    && sed -i '/en_US.UTF-8/s/^# //g' /etc/locale.gen \
    && locale-gen
ENV LANG en_US.UTF-8  
ENV LANGUAGE en_US.UTF-8  
ENV LC_ALL en_US.UTF-8
```
