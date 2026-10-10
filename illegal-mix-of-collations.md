---
title: "JDBC错误Illegal mix of collations解决方法"
source: https://www.boris1993.com/illegal-mix-of-collations.html
date: 2019-08-22
updated: 2026-10-10
tags: [Java, JDBC]
categories: [学知识]
---

# JDBC错误Illegal mix of collations解决方法

最近项目开发时遇到了一个错误，内容是  
`Illegal mix of collations (utf8_unicode_ci,IMPLICIT) and (utf8_general_ci,IMPLICIT) for operation '='`  
经查是关联查询时两个表的排序规则(collation)不同导致的。

解决方法也很简单，有两种方法：

一种是在查询的SQL中指定另一张表的collation，这是一种权宜之计，治标不治本。代码是这样的：

```sql
-- 假设左表的排序规则是 utf8_general_ci,
-- 而右表的排序规则是 utf8_unicode_ci
SELECT
    `a`.`column1`,
    `b`.`column2`
FROM `table_1` AS `a`
LEFT JOIN `table_2` AS `b`
    ON `a`.`id` = `b`.`id`
    COLLATE `utf8_general_ci`
```

另一种方法是修改表和列的排序规则，将其统一。这个方法从根本上解决了这个问题。操作方法可以参考[在 MySQL 中修改表和列的排序规则](https://www.boris1993.com/database/mysql/mysql-convert-tables-and-columns-collation.html) 。
