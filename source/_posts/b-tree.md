---
title: B 树
date: 2020-09-10 19:29:43
tags:
  - data structure
  - tree
  - storage engine
  - index
---

## 概述

**B-Tree** 属于**自平衡树**的一种。其他种类的自平衡树，比如，**AVL**、**Red-Black Tree** 等，都是假设所有数据均保存在内存中。而 **B-Tree** 是用于内存无法保存所有数据的超大数据量场景。

当 **key** 的数量庞大，需要以 **block** 的形式从磁盘中读取数据时，相较于从内存中读取数据而言，访问磁盘需要很长的时间。**B-Tree** 的主要目标，或者说核心思想就是为了减少访问磁盘的次数。

## 时间复杂度

|ALGORITHM|TIME COMPLEXITY|
|:--------|:--------------|
|Search   |O(log n)       |
|Insert   |O(log n)       |
|Delete   |O(log n)       |

> **n**: **B-Tree** 的节点总数。

## 性质

1. 所有的叶子节点在同一级。
2. **B-Tree** 的 **度(degree)** 取决于磁盘块大小。
3. 除根节点之外，其余节点必须至少包含 **t-1** 个 **key**；跟节点至少包含 **1** 个 **key**。
4. 包括根节点在内，所有节点至多包含 **2t-1** 个 **key**。
5. 节点的子节点数量等于节点中 **key** 的数量 **+1**。
6. 节点的所有 **key** 按照升序排列，在 **key: k1, k2** 之间的所有子节点包含 **[k1, k2]** 范围内的所有 **key**。
7. 不同于 **BST** 向下生长，向下收缩。**B-Tree** 从根节点开始生长和收缩。

### 一个简单的例子

![01.png](/images/b-tree/01.PNG)

在上面的例子中，可以观察到:

* 所有的叶子结点均处于同一级别
* 所有非叶子节点都没有空的子树
* 所有非叶子节点 **key** 的数量比其自带数 **少 1**

## 参考阅读

* [Introduction of B-Tree](https://www.geeksforgeeks.org/introduction-of-b-tree-2)
