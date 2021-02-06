---
title: 删除所有 Evicted 状态的 Pod
date: 2020-12-01 13:53:47
tags:
  - kubernetes
---

``` bash
kubectl get pods --all-namespaces -ojson | jq -r '.items[] | select(.status.reason!=null) | select(.status.reason | contains("Evicted")) | .metadata.namespace + " " + .metadata.name' | xargs -n2 -l bash -c 'kubectl delete pods -n $0 $1'
```
