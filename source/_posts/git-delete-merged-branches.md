---
title: 删除已合并的分支
date: 2020-12-31 11:05:56
tags:
  - git
---

``` bash
git branch --merged | egrep -v "(^\*|master|dev)" | xargs git branch -d
```
