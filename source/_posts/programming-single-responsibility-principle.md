---
title: 单一职责原则
date: 2021-02-08 10:06:13
tags:
  - programming
---

> A class or module should have a single responsibility.

单一职责原则要求，一个类或模块只应该有唯一的职责。

首先来明确概念，职责可以被看做一个类或者模块被修改的原因。那么，由这个概念可以得出，当一个类或者模块被修改的原因有两个或以上时，应该考虑将这些功能拆分到多个类或者模块中，从而使每一个类或者模块被修改的原因只有一个。

那么，问题来了: 什么叫做单一职责，或者说，以类为例，如何判断一个类的职责是否单一？这个原则在字面上是非常简单的，但实际上却又非常主观。

## 举个例子

大多数业务中都会涉及到用户信息，例如:

``` go
type UserInfo struct {
    ID                int64
    Name              string
    Email             string
    Telephone         string
    CreateTime        time.Time
    LastLoginTime     time.Time
    AvatarURL         string
    ProvinceOfAddress Province
    CityOfAddress     City
    RegionOfAddress   Region
    DetailedAddress   string
}
```

大部分的开发者会使用如上扁平化的数据结构，事实上也不会有什么问题。但本着从学术的角度出发，来审视一下 `UserInfo` 中的职责有哪些:

* 调用接口时，作为唯一标志
  * `ID`
* 展示作用
  * `Name`
  * `CreateTime`
  * `LastLoginTime`
  * `AvatarURL`
* 认证作用
  * `Email`
  * `Telephone`
* 地址信息
  * `ProvinceOfAddress`
  * `CityOfAddress`
  * `RegionOfAddress`
  * `DetailedAddress`

结果分析下来，竟然包含了四种职责。并且，这四种职责基本上是相互独立的，即任何一种职责的信息发生变化，基本不会影响其他职责的功能。

同时，思考如下几个问题:

1. 用户登录的时候，是直接使用 `UserInfo` 类型做处理，还是使用只包含认证相关字段的类型 `Credential` 呢
2. 展示用户信息的时候，是直接使用 `UserInfo` 类型做处理，还是使用只包含展示信息字段的类型 `UserDisplayInfo` 呢
3. 在处理订单收货地址是，是直接使用 `UserInfo` 类型做处理，还是使用只包含地址相关字段的类型 `Address` 呢
4. 在计算发货地与收货地距离时，也要使用 `UserInfo` 类型处理吗

如此看来，应该将 `UserInfo` 修改为:

``` go
type Credential struct {
    Email             string
    Telephone         string
}

type UserDisplayInfo struct {
    Name              string
    CreateTime        time.Time
    LastLoginTime     time.Time
    AvatarURL         string
}

type Address struct {
    ProvinceOfAddress Province
    CityOfAddress     City
    RegionOfAddress   Region
    DetailedAddress   string
}

type UserInfo struct {
    ID              int64
    Credential      Credential
    UserDisplayInfo UserDisplayInfo
    Address         Address
}
```

但是，还有可能你开发的产品没有订单功能，这时候还有必要有 `Address` 类吗？

因此，开篇才会说，这是一个主观的设计原则。职责是否单一还应该取决于实际的业务场景。

## 参考资料

* [OODesign: single responsibility principle](https://www.oodesign.com/single-responsibility-principle.html)
