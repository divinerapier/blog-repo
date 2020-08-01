---
title: 优化 WHERE 子句
date: 2020-08-01 15:26:44
tags:
  - mysql
  - optimization
  - where clause
---

以 `SELECT` 语句为例，介绍如何优化 `WHERE` 子句。这些优化方法同样适用于 `DELETE` 和 `UPDATE` 语句中的 `WHERE` 子句。

在编写 `SQL` 时，开发者在主观上为了使语句执行的更快而去做一些所谓的“优化”。但实际上，这些“优化”要么会使 `SQL` 失去了可读性，要么是在重复 `MySQL` 做的事情。

下面列举一些 `MySQL` 会做的优化:

* 删除不必要的括号

    ``` SQL
    ((a AND b) AND c OR (((a AND b) AND (c AND d))))
    -> (a AND b AND c) OR (a AND b AND c AND d)
    ```

* 常量替换(Constant folding)

    ``` sql
    (a<b AND b=c) AND a=5
    -> b>5 AND b=c AND a=5
    ```

* 删除恒定条件

    ``` sql
    (b>=5 AND b=5) OR (b=6 AND 5=5) OR (b=7 AND 5=6)
    -> b=5 OR b=6
    ```

    在 `MySQL 8.0.14` 和更高版本中，这是在准备过程中发生的，而不是在优化阶段发生的，这有助于简化联接。 有关更多信息和示例，请参见 [Section 8.2.1.9, “Outer Join Optimization”](https://dev.mysql.com/doc/refman/8.0/en/outer-join-optimization.html)。

* 索引使用的常量表达式仅计算一次。

* 从 `MySQL 8.0.16` 开始，数值类型的列与常数比较时，折叠(folded)或删除无效或越界的值：

    ``` sql
    -- CREATE TABLE t (c TINYINT UNSIGNED NOT NULL);
    SELECT * FROM t WHERE c ≪ 256;
    -≫ SELECT * FROM t WHERE 1;
    ```

* 对于使用`MyISAM` 和 `MEMORY`存储引擎的表，在单一表上执行 [`COUNT(*)`](https://dev.mysql.com/doc/refman/8.0/en/aggregate-functions.html#function_count) 操作时，如果没有 `WHERE` 子句，或者 `WHERE` 子句的表达式 `NOT NULL`，都将直接从表信息中读取。

    > COUNT(*) on a single table without a WHERE is retrieved directly from the table information for MyISAM and MEMORY tables. This is also done for any NOT NULL expression when used with only one table.

* 尽早检测无效的常量表达式。`MySQL` 检测到无效 `SELECT` 语句时，直接返回无结果。

* 不使用 `GROUP BY` 或聚合函数(`COUNT()，MIN()`)时，会把 `HAVING` 与 `WHERE` 合并。

* 对于联接查询中的每个表，构造一个简单的 `WHERE` 实现快速 `WHERE` 评估，尽快跳过行的目的。

* 优先读取常量表。满足以下任意一个即为常量表：

  * 空表或具有一行的表。
  * 使用 `WHERE` 语句构建得到的表，且 `WHERE` 子句中的所有列只能是 `PRIMARY KEY` 或 `NOT NULL UNIQUE` 索引与常量表达式比较。

  以下所有表均用作常量表：

    ``` sql
    WHERE primary_key=1;

    WHERE t1.primary_key=1 AND t2.primary_key=t1.id;

    -- PRIMARY KEY (column1,column2)
    WHERE column1=5 AND column2=7

    -- unique_not_null_column INT NOT NULL UNIQUE
    WHERE unique_not_null_column=5
    ```
  
  参考 [7.2.1.4 Constants and Constant Tables](https://dev.mysql.com/doc/internals/en/optimizer-constants-constant-tables.html) 了解更多有关常量与常量表。

* 通过尝试所有可能的方法，找到用于联接表的最佳联接组合。如果 `ORDER BY` 和 `GROUP BY` 子句中的所有列都来自同一表，则在连接时优先使用该表。

* 如果 `ORDER BY` 子句和 `GROUP BY` 子句使用不同的列，或者 `ORDER BY`/`GROUP BY` 使用的列不属于联接队列中第一个表，则会创建一个临时表。

  > If there is an ORDER BY clause and a different GROUP BY clause, or if the ORDER BY or GROUP BY contains columns from tables other than the first table in the join queue, a temporary table is created.

* 如果使用 `SQL_SMALL_RESULT` 修饰符，`MySQL` 将使用内存临时表。

* 从表的所有索引中选择一个最佳索引使用。或者，优化器认为全表扫描更有效时，会选择全表扫描。曾经，当使用最佳索引仍然会跨越表的 `30%`(`spanned more than 30% of the table`)时，就会选择使用全表扫描。但现在，使用索引还是全表扫描不再只取决于固定百分比，同时还要考虑其他因素，例如表大小，行数和 `I/O` 块大小。

* 在某些情况下，`MySQL` 可以从索引中读取数据行，而无需查询数据文件。如果索引中使用的所有列都是数字列，则仅使用索引树解析查询。

* 在输出每一行之前，将跳过与 `HAVING` 子句不匹配的行。
