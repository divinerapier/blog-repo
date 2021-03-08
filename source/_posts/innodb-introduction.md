---
title: InnoDB 入门
date: 2021-03-08 13:27:56
tags:
  - mysql
  - innnodb
---

`InnoDB` 是一种平衡可靠性与性能的通用存储引擎。在 `MySQL 8.0` 中，通过 `CREATE TABLE` 语句创建表时，如若未通过 `ENGINE` 子句指定引擎，将采用 `InnoDB` 作为默认的存储引擎。

## InnoDB 的关键优势

* `InnoDB` 的 `DML` 操作遵循 `ACID` 模型，且具有提交、回滚、错误恢复等事务的功能与能力，保障用户数据安全。
* 行级锁定和 `Oracle` 风格的 **一致性读取** 提高了多用户使用时的并发性与性能。
* `InnoDB` 表将数据存放在磁盘上，基于 **主键** 来优化查询。每个 `InnoDB` 表都有一个称为 **聚集索引** 的主键索引，使用该索引组织数据可以将查找主键的 `I/O` 最小化。
* 为了维护数据完整性，`InnoDB` 支持外键约束。当使用外键时，`InnoDB` 会检查插入、更新和删除等语句，来确保它们不会导致相关表之间的数据不一致。

### InnoDB 功能列表

|Feature|Support|
|:------|:------|
|B-tree indexes|Yes|
|Backup/point-in-time recovery (在服务端实现，而非存储引擎。)|Yes|
|Cluster database support|No|
|Clustered indexes|Yes|
|Compressed data|Yes|
|Data caches|Yes|
|Encrypted data|Yes (在服务器端通过加密功能实现;在MySQL 5.7和更高版本中，支持数据静止加密。)|
|Foreign key support|Yes|
|Full-text search indexes|Yes (在MySQL 5.6和更高版本中支持全文索引。)|
|Geospatial data type support|Yes|
|Geospatial indexing support|Yes (MySQL 5.7和更高版本中提供了对地理空间索引的支持。)|
|Hash indexes|No (InnoDB内部利用哈希索引来实现自适应哈希索引特性。)|
|Index caches|Yes|
|Locking granularity|Row|
|MVCC|Yes|
|Replication support (在服务端实现，而非存储引擎。)|Yes|
|Storage limits|64TB|
|T-tree indexes|No|
|Transactions|Yes|
|Update statistics for data dictionary|Yes|

## 使用 InnoDB 的优势

使用 `InnoDB` 有如下优势:

* 如果服务器由于硬件或软件问题而意外退出，无论崩溃时数据库内部遇到什么问题，在重新启动数据库后都不需要做任何特殊的操作。`InnoDB` 会恢复崩溃前已经确定的更改，并撤销正在进行但未提交的更改，允许用户重新启动并从停止的地方继续。
* `InnoDB` 存储引擎维护其内部的缓冲池，当数据被访问时，该缓冲池会在内存中缓存表和索引数据。频繁被使用到的数据将直接从内存中处理。多种类型的信息都可以通过使用该缓存来提高处理速度。在专用的数据库服务器上，通常会将高达 `80%` 的物理内存分配给缓冲池使用。
* 可以设置外键，保证多个相关数据表之间的数据完整性。
* 如果磁盘或内存中的数据损坏，**校验和** 机制会在使问题假数据之前向用户发出警告。变量 `innodb_checksum_algorithm` 定义了 `InnoDB` 使用了何种校验和算法。
* 在 `WHERE`、`ORDER BY`、`GROUP BY` 等子句与 `JOIN` 操作中使用主键时，`InnoDB` 将自动优化涉及这些列的操作，使这些操作的执行速度非常快。
* 更改缓冲(change buffering) 这一自动极致会对插入，更新和删除进行优化。 `InnoDB` 不仅允许对同一表的并发读写访问，而且会缓存被更改数据，使磁盘 `I/O` 流水线化。
* 性能优势并不仅限于哪些需要很长时间查询的大表。当表中的某些被反复访问时，`InnoDB` 会通过自适应哈希索引( Adaptive Hash Index) 处理，加快查询速度，达到类似使用哈希表查找的效果。
* 允许压缩表和关联索引。
* 加密数据。
* 在线创建和删除索引，执行其他 `DDL` 操作对性能和可用性有较小的影响。
* 截断每个表文件的表空间的速度非常快，被释放的磁盘空间不仅可以供 `InnoDB` 使用，操作系统同样可以重用。
* 表数据的存储布局对于使用动态(DYNAMIC) 行格式的 `BLOB` 和长文本字段更有效。
* 可以通过查询 `INFORMATION_SCHEMA` 表来监视存储引擎的内部工作。
* 可以通过查询 `performance_schema` 表来监控存储引擎的性能详情。
* 可以混合使用 `InnoDB` 表和使用其他 `MySQL` 存储引擎的表，即使是在同一个语句中。例如，可以使用 `JOIN` 操作在单个查询中组合来自 `InnoDB` 和内存表的数据。
* `InnoDB` 的设计目标是在处理大数据量时的提高 `CPU` 效率并达到最高性能。
* `InnoDB` 表可以处理大量数据，即使在文件大小被限制为 `2GB` 的操作系统上。

## InnoDB 表的最佳实践

在使用 `InnoDB` 的表时有如下最佳实践:

* 使用查询最频繁的一列或多列作为表的主键，如果没有明显的主键，则使用自动递增 `ID`。

## 参考资料

* [Introduction to InnoDB](https://dev.mysql.com/doc/refman/8.0/en/innodb-introduction.html)
* [Benefits of Using InnoDB Tables](https://dev.mysql.com/doc/refman/8.0/en/innodb-benefits.html)
* [Best Practices for InnoDB Tables](https://dev.mysql.com/doc/refman/8.0/en/innodb-best-practices.html)
