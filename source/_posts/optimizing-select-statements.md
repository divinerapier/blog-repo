---
title: 优化 SELECT 语句
date: 2020-07-28 21:45:48
tags:
  - mysql
  - optimization
  - select statements
---

数据库中所有查找操作，均以 `SELECT` 语句的形式执行。无论是实现网站秒级以内的响应时间，还是期望在生成大量的隔夜报告是缩短数小时的执行时间，调试这类语句都是重中之重。

除了 `SELECT` 语句外，相同的技术还适用于诸如 `CREATE TABLE ... AS SELECT`，`INSERT INTO ... SELECT` 和 `DELETE` 语句中的 `WHERE` 子句。但由于这些语句同时会涉及到读写两种操作，因此还需要考虑其他方面的性能问题。

多节点集群支持 **`JOIN` 查询下推优化(`join pushdown optimization`)**，能将符合条件的 `JOIN` 完整地发送到集群的数据节点，让这个查询请求被分发到这些节点上并行执行。有关此优化的更多信息，请参见[Conditions for NDB pushdown joins](https://dev.mysql.com/doc/refman/8.0/en/mysql-cluster-options-variables.html#ndb_join_pushdown-conditions)。

优化查询的核心因素：

* 如何查询语句 `SELECT ... WHERE` 执行的非常之慢，首选提速方法就是检查是否可以添加[索引](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_index)。在 `WHERE` 子句中使用的列上设置索引，以加快执行，过滤，检索结果等操作的速度。同时，索引信息需要占用一定的磁盘空间，请尽可能在有一定关联性的查询中复用索引。

    在执行 `JOIN` 查询，外键关联等需要多个表参与的查询语句时，索引尤为重要。此时，可以通过 [EXPLAIN](https://dev.mysql.com/doc/refman/8.0/en/explain.html) 语句来确定，执行 `SELECT` 语句时，实际有哪些索引真正被使用了。参见 [Section 8.3.1, “How MySQL Uses Indexes”](https://dev.mysql.com/doc/refman/8.0/en/mysql-indexes.html) and [Section 8.8.1, “Optimizing Queries with EXPLAIN”](https://dev.mysql.com/doc/refman/8.0/en/using-explain.html)。

* 隔离和调整查询中花费时间过多的任何部分，例如函数调用。 根据查询的结构方式，可以对结果集中的每一行调用一次函数，甚至可以对表中的每一行调用一次函数，从而极大地提高了效率。

* 最小化查询中[全表扫描](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_full_table_scan)的次数，特别是对于大表。

* 通过定期使用 [`ANALYZE TABLE`](https://dev.mysql.com/doc/refman/8.0/en/analyze-table.html) 语句使表统计信息保持最新状态，让优化器具有充足的信息构造有效执行计划。

* 了解每种存储引擎的调整技术，索引技术和配置参数。无论是 `InnoDB` 还是 `MyISAM`，都各自具有一套保持查询高性能的准则。参见 [Section 8.5.6, “Optimizing InnoDB Queries”](https://dev.mysql.com/doc/refman/8.0/en/optimizing-innodb-queries.html) 和 [Section 8.6.1, “Optimizing MyISAM Queries”](https://dev.mysql.com/doc/refman/8.0/en/optimizing-queries-myisam.html)。

* 使用 [Section 8.5.3, “Optimizing InnoDB Read-Only Transactions”](https://dev.mysql.com/doc/refman/8.0/en/innodb-performance-ro-txn.html) 中的技术优化 `InnoDB` 表的单查询事务。

* 避免以难以理解的方式转换查询，特别是在优化器自动执行某些相同转换的情况下。

* 当使用基本准则不能轻松解决性能问题时，可以通过阅读 [EXPLAIN](https://dev.mysql.com/doc/refman/8.0/en/explain.html) 计划并调整索引，`WHERE`，`JOIN` 等子句来调查特定查询的内部详细信息。(有一定经验之后，阅读 [EXPLAIN](https://dev.mysql.com/doc/refman/8.0/en/explain.html) 计划可能是每个查询的第一步。)

* 调整 `MySQL` 用于缓存的内存区域的大小和属性。合理使用 InnoDB 的[buffer pool](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_buffer_pool)，`MyISAM` 的 `key cache` 和 `MySQL` 的 `query cache`，可以让重复查询的运行速度更快。

* 即使查询已经使用了内存缓存，也依然可能对其进一步优化，使其需要更少的内存，从而使应用程序更具可伸缩性。比如，应用程序可以同时处理更多的用户，更大的请求量，而不会导致性能大幅下降。

* 锁定问题，多个会话同时访问同一张表可能会影响查询速度。

### 更多优化策略

* [WHERE Clause Optimization](https://blog.divinerapier.cn/2020/08/01/where-clause-optimization/)
* Range Optimization
* Index Merge Optimization
* Hash Join Optimization
* Engine Condition Pushdown Optimization
* Index Condition Pushdown Optimization
* Nested-Loop Join Algorithms
* Nested Join Optimization
* Outer Join Optimization
* Outer Join Simplification
* Multi-Range Read Optimization
* Block Nested-Loop and Batched Key Access Joins
* Condition Filtering
* Constant-Folding Optimization
* IS NULL Optimization
* ORDER BY Optimization
* GROUP BY Optimization
* DISTINCT Optimization
* LIMIT Query Optimization
* Function Call Optimization
* Window Function Optimization
* Row Constructor Expression Optimization
* Avoiding Full Table Scans
