---
title: 在 Archlinux 使用 diff cmp
date: 2020-12-16 14:59:09
tags:
  - archlinux
---

在 [kubernetes](https://github.com/kubernetes/kubernetes) 仓库中的脚本依赖于 `cmp & diff` 命令。

这些命令在 [diffutils](https://www.archlinux.org/packages/core/x86_64/diffutils/files/) 中。

使用如下方式安装:

``` bash
sudo pacman -S diffutils
```
