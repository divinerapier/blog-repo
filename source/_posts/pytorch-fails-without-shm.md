---
title: 未设置 shm 导致 PyTorch 任务失败
date: 2021-03-05 10:01:27
tags:
  - pytorch
  - docker
  - k8s
  - linux
---

最近，配合研发同学将原来在老集群上直接用 `docker` 运行的 `pytorch` 算法迁移到使用 `k8s` 的新集群上运行。结果，很不幸，研发同学说无法运行。错误日志如下:

``` bash
10:59:01[asr.utils.bootstrap]-ERROR-Traceback (most recent call last):
10:59:01[asr.utils.bootstrap]-ERROR-  File "/miniconda3/envs/pyasr/lib/python3.7/runpy.py", line 193, in _run_module_as_main
10:59:01[asr.utils.bootstrap]-ERROR-"__main__", mod_spec)
10:59:01[asr.utils.bootstrap]-ERROR-  File "/miniconda3/envs/pyasr/lib/python3.7/runpy.py", line 85, in _run_code
10:59:01[asr.utils.bootstrap]-ERROR-exec(code, run_globals)
10:59:01[asr.utils.bootstrap]-ERROR-  File "/pytorch-asr/egs/chn_40h/extend_code/launch.py", line 29, in <module>
10:59:01[asr.utils.bootstrap]-ERROR-main()
10:59:01[asr.utils.bootstrap]-ERROR-  File "/pytorch-asr/asr/launch.py", line 10, in main
10:59:01[asr.utils.bootstrap]-ERROR-launch(args)
10:59:01[asr.utils.bootstrap]-ERROR-  File "/pytorch-asr/asr/cli/launch.py", line 148, in launch
10:59:01[asr.utils.bootstrap]-ERROR-trainer.train_on(data)
10:59:01[asr.utils.bootstrap]-ERROR-  File "/pytorch-asr/asr/trainer/trainer.py", line 77, in train_on
10:59:01[asr.utils.bootstrap]-ERROR-self.train_epoch(data['tr'])
10:59:01[asr.utils.bootstrap]-ERROR-  File "/pytorch-asr/asr/trainer/trainer.py", line 41, in train_epoch
10:59:01[asr.utils.bootstrap]-ERROR-return self._one_epoch(data_queue, is_training=True)
10:59:01[asr.utils.bootstrap]-ERROR-  File "/pytorch-asr/asr/trainer/trainer.py", line 63, in _one_epoch
10:59:01[asr.utils.bootstrap]-ERROR-return self.one_epoch(data_queue, is_training)
10:59:01[asr.utils.bootstrap]-ERROR-  File "/pytorch-asr/asr/trainer/naive_trainer.py", line 115, in one_epoch
10:59:01[asr.utils.bootstrap]-ERROR-for batch_idx, batch in enumerate(self.timer['io'].profile(data_queue)):
10:59:01[asr.utils.bootstrap]-ERROR-  File "/pytorch-asr/asr/trainer/common.py", line 102, in profile
10:59:01[asr.utils.bootstrap]-ERROR-value = next(iterator)
10:59:01[asr.utils.bootstrap]-ERROR-  File "<string>", line 2, in get
10:59:01[asr.utils.bootstrap]-ERROR-  File "/miniconda3/envs/pyasr/lib/python3.7/multiprocessing/managers.py", line 834, in _callmethod
10:59:01[asr.utils.bootstrap]-ERROR-raise convert_to_error(kind, result)
10:59:01[asr.utils.bootstrap]-ERROR-_queue
10:59:01[asr.utils.bootstrap]-ERROR-.
10:59:01[asr.utils.bootstrap]-ERROR-Empty
11:12:55[asr.utils.bootstrap]-WARNING-Version 0.3.1.dev28
```

看到这个错误，就挺迷茫的。

根据错误信息:

``` bash
10:59:01[asr.utils.bootstrap]-ERROR-  File "/pytorch-asr/asr/trainer/naive_trainer.py", line 115, in one_epoch
10:59:01[asr.utils.bootstrap]-ERROR-for batch_idx, batch in enumerate(self.timer['io'].profile(data_queue)):
10:59:01[asr.utils.bootstrap]-ERROR-  File "/pytorch-asr/asr/trainer/common.py", line 102, in profile
10:59:01[asr.utils.bootstrap]-ERROR-value = next(iterator)
```

推断是读数据的时候出现了问题。虽然看起来与 `GPU` 没有关系，但是，为了保险起见，依然对比了前后环境的 `NVIDIA Driver`，`CUDA` 等版本信息，结果是一致的。而且，`NCCL` 是在镜像中安装的，不太可能有问题。

之后，尝试在容器中安装 `perf`，但是失败了，尝试在容器中使用 `strace`，提示没有权限。

正在我继续死磕的时候，研发提供了另一段错误日志:

``` bash
pytorch-956sg:9319:9379 [0] include/shm.h:48 NCCL WARN Error while creating shared memory segment nccl-shm-recv-183375136c5888b6-0-2-3 (size 9637888)
```

这个错误就很有价值啊，创建共享内存失败。

此前，反复与研发同学确认过，是否只依赖于外部的 `NVIDIA Driver` 与 `CUDA`，得到了肯定的答复。因此，就一直以此为前提进行排查。但现在我产生了怀疑，问研发要来启动命令，隐藏敏感数据后如下:

``` bash
docker run -dit /data:/data --name asr --ipc=host image:latest /bin/bash
```

果然就发现了隐藏信息: **--ipc=host** 。

因此，修改在原有创建任务的 `yaml` 文件中增加 `volume`:

``` yaml
   spec:
     containers:
       - name: asr
         image: image:latest
         volumeMounts:
           - mountPath: /dev/shm
             name: shm
     volumes:
       - name: shm
         emptyDir:
            medium: Memory
```

在容器内，挂载一个 `tmpfs` 到 `/dev/shm`。成功解决上述两个问题。

## 总结

1. 虽然成功解决了两个问题，但是对于第一个问题还是感到迷惑。虽然，根据答案反推原因，可以查到 `DataLoader` 在工作的时候会依赖于 `shm`。但是，查到的错误却不同于第一份错误日志。

2. 这次是运气好，遇到了第二个错误明显的提示。否则，可能就需要继续跟 `perf`，`strace` 作斗争了，甚至于调试 `coredump`。

3. 在遇到错误二之前，已经决定自己在新集群上使用 `docker` 运行看看了，其实，这样的话也能发现被隐藏的 `--ipc=host`。

4. 最终要的一点，不要轻易相信他人给定的条件。即使对方不是有意隐瞒，也可能会由于遗忘，或本来就不清楚等各种原因，导致丢失已知条件。
