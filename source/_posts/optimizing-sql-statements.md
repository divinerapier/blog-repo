---
title: 优化SQL语句
date: 2020-07-28 19:41:54
tags:
  - mysql
  - optimization
---

1. Optimizing SELECT Statements
2. Optimizing Subqueries, Derived Tables, View References, and Common Table Expressions
3. Optimizing INFORMATION_SCHEMA Queries
4. Optimizing Performance Schema Queries
5. Optimizing Data Change Statements
6. Optimizing Database Privileges
7. Other Optimization Tips

数据库应用程序的核心逻辑是通过 `SQL` 语句执行的，无论这些 `SQL` 语句是通过解释程序直接发出还是通过调用 `API` 提交。之后提到的调整准则有助于加快各种使用到 `MySQL` 应用程序的速度。指南涵盖了读写数据的 `SQL` 操作，通用 `SQL` 操作的底层开销以及在特定方案（例如数据库监视）中使用的操作。
