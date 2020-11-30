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
echo "set fileencodings=utf-8,ucs-bom,gb18030,gbk,gb2312,cp936" >> ~/.vimrc
echo "set termencoding=utf-8" >> ~/.vimrc
echo "set encoding=utf-8" >> ~/.vimrc
```

## 在 Dockerfile 中处理

``` dockerfile
RUN apt update --fix-missing \
    && apt install -y locales \
    && sed -i '/en_US.UTF-8/s/^# //g' /etc/locale.gen \
    && locale-gen

RUN echo "set fileencodings=utf-8,ucs-bom,gb18030,gbk,gb2312,cp936" >> ~/.vimrc \
    && echo "set termencoding=utf-8" >> ~/.vimrc \
    && echo "set encoding=utf-8" >> ~/.vimrc

ENV LANG en_US.UTF-8
ENV LANGUAGE en_US.UTF-8
ENV LC_ALL en_US.UTF-8
```
