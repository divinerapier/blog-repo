---
title: 针对接口编程
date: 2021-02-06 16:22:31
tags:
  - programming
---

> Program to an interface, not an implementation.

针对接口，而非实现编程 —— 出自 GoF《设计模式: 可复用面向对象软件基础》。**接口** 表示一组 **协议** 或 **约定**，是一组功能的集合。

接口在不同的语言中，有不同的称呼:

* **rust** 中的 **trait**
* **go** 中的 **interface**
* **swift** 中的 **protocol**
* **java** 中的 **interface**

但无论他们的名字是否相同，其作用都是表达一组 **抽象的功能集合**。

以上字面意思很好理解，下面来详细说明。

## 为什么要针对接口编程

初级开发者在实现某一具体需求时，其思考的关注点在于需求本身，分析实现这个需求要经过哪些步骤，每个步骤又要做那些事情，以此类推。在经过逐层分析，想明白流程之后，这名开发者就会把解决步骤转化为代码。

如果只看这个需求，这名开发者完成并交付了这个功能。但是，当需求发生变动时，需要修改功能，或增加功能时，由于上面的实现方式是面向过程的，步骤与步骤之间互相暴露实现的细节。因此，当修改了某一个步骤的代码时，就非常可能需要同步修改前后相邻步骤的代码，最坏的情况可能要修改全部的代码。结果就是不小的开发工作量与测试工作量。

如果开发者使用针对接口编程的方式，封装不稳定的实现细节，暴露稳定的接口。当需求发生变化时，只需要修改相关的接口实现，而因为接口不变，所以无需改动其他代码。

## 举个例子

产品给你分配了一个任务: 有一批保存在 `aws s3` 上的数据，需要你将它们同步到 `aliyun oss` 上。

### 针对实现编程

如果是使用针对实现编程方式，那么在收到这个需求之后，应该就会思考: 把 `aws s3` 的数据同步到 `aliyun oss` 上，步骤差不多是:

1. 想办法把数据从 `aws s3` 上下载到磁盘或者内存
2. 把磁盘或者内存中的数据上传到 `aliyun oss`

想到这里，感觉差不多可以写代码了:

``` go
type AwsS3 struct {}
type AliyunOSS struct {}

type (s3 *AwsS3) Sync(oss *AliyunOSS, names []string) {
  for _, name := ranges names {
    data := s3.Download(name)
    oss.Upload(name, data)
  }
}
```

经过测试确认无误后，发布上线，成功运行了几个月没出现 Bug。

#### 需求变更

后来呢，产品要求你有的数据要同步到 `aliyun oss`，有一些要同步到 `ceph s3` 上。你一看，这个需求简单啊，我会。然后，你把上面的代码复制了一份，稍加改动:

``` go
type CephS3 struct {}

type Meta struct {
  Name     string
  ToOss    bool
  ToCephS3 bool
}

type (s3 *AwsS3) syncToOss(oss *AliyunOSS, name string) {
  data := s3.Download(name)
  oss.Upload(name, data)
}

type (s3 *AwsS3) syncToCephS3(cephs3 *CephS3, name string) {
  data := s3.Download(name)
  cephs3.Upload(name, data)
}

type (s3 *AwsS3) Sync(oss *AliyunOSS, cephs3 *CephS3, metas *Meta) {
  for _, meta := ranges metas {
    data := s3.Download(meta.Name)
    if meta.ToOss {
      s3.syncToOss(oss, meta.Name)
    }
    if meta.ToCephS3 {
      s3.syncToCephS3(cephs3, meta.Name)
    }
  }
}
```

不到一天的时间就完成了开发、测试，第二天就上线了。

### 针对接口编程

实现将数据从 `aws s3` 同步到 `aliyun oss` 上，可以提取到的信息包括 `aws s3`、`aliyun oss` 两种存储系统，同步是要实现的操作。

由此引发思考，同步是一种动作，未来发生变化的可能性不大，或许有增加其他的动作可能，比如对比两个存储系统的内容，但这应该算作扩展需求，暂时无需考虑；但是存储系统就不好说了，现在是这样的要求，未来可能就会要求同步到 `minio` 上，或者同步到另一个 `aws s3` 上。

至此，可以考虑将存储系统抽象为一组接口:

``` go
type Storage interface {
  Upload(name string, data []byte)
  Download(name string) []byte
}
```

再来思考，现在是要求从 `aws s3` 同步到 `aliyun oss` 上，既然上面已经认为未来有很大概率需要同步其他存储系统，由此提出问题: 如何确定同步的目标系统？先大胆猜测:

1. 一个服务只负责一种特定源到特定目标的同步任务
1. 在创建同步任务时指定，即在请求参数中，适用于本次任务的所有数据
1. 根据某种策略，或者是算法，确定每一个文件的源与目的分别是哪里

那么接下来逐条分析:

* 方法一，极其不灵活，如果有 **n** 个存储系统，那么一共需要启动 **nx(n-1)**个服务，无论是对于使用者，还是维护者来说，都可谓是灾难
* 方法二，相比于方法一，极大地提高了灵活性，只需要一个服务就能替代上述 **nx(n-1)** 个服务
* 方法三，相比于方法二，更进一步提高了灵活性，每一个文件都可以有独立的源与目标。但这种方法只是看起来很美好，很灵活，实际上可能并没有真实的使用场景。原因是，如果每一个文件都可以具有独立的目标，那么完全可以将目标相同的文件聚合到一起，作为一个批量的任务，这样就演变为了 **方法二**

综上所述，下面实现 **方法二**:

``` go
type Batch struct {
  From    string
  Targets []string
  Names   []string
}

func Sync(batch *Batch) {
  from := NewStorage(batch.From)
  targets := NewStorages(batch.Targets)
  for _, name := range batch.Names {
    data := from.Download(name)
    for _, target := range targets {
      target.Upload(name, data)
    }
  }
} 
```

到此为止，所有代码都是面向接口编程，需求中提到的 `aws s3` 与 `aliyun oss` 还没有出现:

``` go
type AwsS3 struct {}

func (s3 *AwsS3) Upload(name string, data []byte) {}
func (s3 *AwsS3) Download(name string) []byte {}

type AliyunOSS struct {}

func (s3 *AwsS3) Upload(name string, data []byte) {}
func (s3 *AwsS3) Download(name string) []byte {}
```

然后，需要实现两个函数:

``` go
func NewStorage(name string) Storage {
  switch name {
    case "awss3":
      return &AwsS3{}
    case "aliyunoss":
      return AliyunOSS{}
    default:
      panic("unknown name: %s", name) // 只作为演示 
  }
}

func NewStorages(names string) []Storage {
  var results []Storage
  for _, name := range names {
    results = append(results, NewStorage(name))
  }
  return results
}
```

到此为止，才终于完成了原始需求。使用面向实现方式的同学会说: 你这代码量要比面向实现的方法多很多，你实现了一个需求，我都实现好几个了。

对此，我是不慌的。

当产品要求将数据同步到 `cephs3`，`minio` 时，或者是从 `tencent cos`，本地磁盘同步到其他地方时，`func Sync` 是完全不需要改动的，代码的改动只有:

1. 增加对应的类型实现 **Storage** 接口:

  ``` go
  type CephS3 struct {}

  func (ceph *CephS3) Upload(name string, data []byte) {}
  func (ceph *CephS3) Download(name string) []byte {}

  type MinIO struct {}

  func (minio *MinIO) Upload(name string, data []byte) {}
  func (minio *MinIO) Download(name string) []byte {}

  type Posix struct {}

  func (fs *Posix) Upload(name string, data []byte) {}
  func (fs *Posix) Download(name string) []byte {}

  type TencentOSS struct {}

  func (oss *TencentOSS) Upload(name string, data []byte) {}
  func (oss *TencentOSS) Download(name string) []byte {}
  ```

1. 修改 **func NewStorage**:

``` go
func NewStorage(name string) Storage {
  switch name {
    case "awss3":
      return &AwsS3{}
    case "aliyunoss":
      return AliyunOSS{}
    case "cephs3":
      return CephS3{}
    case "minio":
      return MinIO{}
    case "posix":
      return Posix{}
    case "tencentoss":
      return TencentOSS{}
    default:
      panic("unknown name: %s", name) // 只作为演示 
  }
}
```

就完成所有的修改了。如果是面向实现的方式，算了，我不想了。

## 总结

抽象会提升思维上的难度，但却能提高代码的灵活性。而且，灵活性还会随着抽象程度的提升一起提高。优秀的代码设计，不但能满足于眼前的需求，还提供了在不改变已有设计的前提支持对未来可能变化的需求的能力。
