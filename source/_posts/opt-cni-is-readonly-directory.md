---
title: 目录 /opt/bin 是只读目录 
date: 2020-10-26 11:44:37
tags:
  - kubernetes
  - faq
---

在部署 [Canal](https://docs.projectcalico.org/getting-started/kubernetes/flannel/flannel) 时遇到如下错误:

1. 查看 **Pod** 状态:

  ``` bash
  $ kubectl get pods --all-namespaces

  NAMESPACE     NAME                                                              READY   STATUS                   RESTARTS   AGE
  kube-system   canal-5qk26                                                       0/2     Init:RunContainerError   0          10m
  kube-system   kube-proxy-tt2qn                                                  0/1     CrashLoopBackOff         10         11m
  ```

1. 查看 **canal-5qk26** 事件:

  ``` bash
  $ kubectl -n kube-system describe pod canal-5qk26

  Events:
    Type     Reason          Age                     From                Message
    ----     ------          ----                    ----                -------
    Normal   Scheduled       6m41s                   default-scheduler   Successfully assigned kube-system/canal-5qk26 to ubuntu-01
    Warning  Failed          6m40s                   kubelet, ubuntu-01  Error: failed to start container "install-cni": Error response from daemon: can't join IPC of container 1f3affaa9eba3f1087ac2309f7c4147a54e2cbad09be733e9c394ce7a8ba583b: container 1f3affaa9eba3f1087ac2309f7c4147a54e2cbad09be733e9c394ce7a8ba583b is not running
    Warning  Failed          6m39s                   kubelet, ubuntu-01  Error: failed to start container "install-cni": Error response from daemon: cannot join network of a non running container: 1861c340dfadba101b333af1163329a88a8e02fd25e5d657b4e0954acd09d3d5
    Warning  Failed          6m37s                   kubelet, ubuntu-01  Error: failed to start container "install-cni": Error response from daemon: error while creating mount source path '/opt/cni/bin': mkdir /opt/cni: read-only file system
    Warning  BackOff         6m34s (x3 over 6m38s)   kubelet, ubuntu-01  Back-off restarting failed container
    Normal   Pulled          6m33s (x5 over 6m40s)   kubelet, ubuntu-01  Container image "calico/cni:v3.16.4" already present on machine
    Normal   Created         6m33s (x5 over 6m40s)   kubelet, ubuntu-01  Created container install-cni

    Normal   SandboxChanged  100s (x269 over 6m39s)  kubelet, ubuntu-01  Pod sandbox changed, it will be killed and re-created.
  ```

1. 查看 **kube-proxy-tt2qn** 事件:

  ``` bash
  $ kubectl -n kube-system describe pod kube-proxy-tt2qn

  Events:
    Type     Reason          Age                      From                Message
    ----     ------          ----                     ----                -------
    Normal   Scheduled       11m                      default-scheduler   Successfully assigned kube-system/kube-proxy-tt2qn to ubuntu-01
    Normal   Pulled          11m (x2 over 11m)        kubelet, ubuntu-01  Container image "registry.aliyuncs.com/google_containers/kube-proxy:v1.19.3" already present on machine
    Normal   Created         11m (x2 over 11m)        kubelet, ubuntu-01  Created container kube-proxy
    Warning  Failed          11m                      kubelet, ubuntu-01  Error: failed to start container "kube-proxy": Error response from daemon: cannot join network of a non running container: 119388927b173ad23226c1049db0c1269ada343b133774807a1615cc79442246
    Warning  BackOff         11m (x9 over 11m)        kubelet, ubuntu-01  Back-off restarting failed container
    Normal   SandboxChanged  11m (x10 over 11m)       kubelet, ubuntu-01  Pod sandbox changed, it will be killed and re-created.
  ```

1. 查看 **Worker** 节点上 **Kubelet** 与 **Docker** 日志:

  ``` bash
$ journalctl -f

  -- Logs begin at Mon 2020-10-19 10:09:50 UTC. --
  Oct 26 02:31:35 ubuntu-01 docker.dockerd[1781]: time="2020-10-26T02:31:35.429587085Z" level=error msg="Handler for POST /v1.40/containers/aef879b89cd75d23249e76091a4716c1303855b904979e983920aae02da01d18/start returned error: error while creating mount source path '/opt/cni/bin': mkdir /opt/cni: read-only file system"
  Oct 26 02:31:35 ubuntu-01 kubelet[281110]: E1026 02:31:35.474035  281110 remote_runtime.go:248] StartContainer "aef879b89cd75d23249e76091a4716c1303855b904979e983920aae02da01d18" from runtime service failed: rpc error: code = Unknown desc = failed to start container "aef879b89cd75d23249e76091a4716c1303855b904979e983920aae02da01d18": Error response from daemon: error while creating mount source path '/opt/cni/bin': mkdir /opt/cni: read-only file system
  Oct 26 02:31:35 ubuntu-01 kubelet[281110]: E1026 02:31:35.474175  281110 pod_workers.go:191] Error syncing pod 5f45d450-c4e9-45dc-b2c6-52a95570ba71 ("canal-5qk26_kube-system(5f45d450-c4e9-45dc-b2c6-52a95570ba71)"), skipping: failed to "StartContainer" for "install-cni" with RunContainerError: "failed to start container \"aef879b89cd75d23249e76091a4716c1303855b904979e983920aae02da01d18\": Error response from daemon: error while creating mount source path '/opt/cni/bin': mkdir /opt/cni: read-only file system"
  Oct 26 02:31:36 ubuntu-01 audit[315954]: AVC apparmor="DENIED" operation="exec" info="no new privs" error=-1 profile="snap.docker.dockerd" name="/pause" pid=315954 comm="runc:[2:INIT]" requested_mask="x" denied_mask="x" fsuid=0 ouid=0 target="docker-default"
  Oct 26 02:31:36 ubuntu-01 kernel: audit: type=1400 audit(1603679496.465:6039): apparmor="DENIED" operation="exec" info="no new privs" error=-1 profile="snap.docker.dockerd" name="/pause" pid=315954 comm="runc:[2:INIT]" requested_mask="x" denied_mask="x" fsuid=0 ouid=0 target="docker-default"
  Oct 26 02:31:36 ubuntu-01 kubelet[281110]: W1026 02:31:36.635730  281110 cni.go:239] Unable to update cni config: no networks found in /etc/cni/net.d
  ```

根据上述日志信息，可以确认根本错误出现在 **Docker** 中:

``` text
error while creating mount source path '/opt/cni/bin': mkdir /opt/cni: read-only file system
```

但宿主机目录 **/opt/bin** 的权限为:

``` bash
$ stat /opt/cni/

  File: /opt/cni/
  Size: 4096            Blocks: 8          IO Block: 4096   directory
Device: fd00h/64768d    Inode: 917533      Links: 3
Access: (0755/drwxr-xr-x)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2020-10-26 01:41:07.222221020 +0000
Modify: 2020-10-20 13:00:36.461719609 +0000
Change: 2020-10-20 13:00:36.461719609 +0000
 Birth: -
```

因此，推测为 **Docker** 服务的问题。最终，通过重启 **Docker** 解决问题:

``` bash
sudo systemctl restart docker.service
```
