---
title: "在MySQL中修改表和列的排序规则"
source: https://www.boris1993.com/mysql-convert-tables-and-columns-collation.html
date: 2019-08-22
updated: 2026-10-10
tags: [MySQL, collation]
categories: [小技巧]
---

# 在MySQL中修改表和列的排序规则

使用如下SQL语句即可更新一张表的字符集(character set)和排序规则(collation)：

```sql
-- 此处假设使用utf8字符集，以及使用utf8_unicode_ci排序规则
ALTER TABLE `table_name` CONVERT TO CHARACTER SET utf8 COLLATE utf8_unicode_ci;
```

然后可以使用如下SQL查询表和列的字符集和排序规则是否修改成功：

```sql
-- 查询表的信息
SELECT `TABLE_SCHEMA`, `TABLE_NAME`, `TABLE_COLLATION`
FROM `information_schema`.`TABLES`
WHERE `TABLE_NAME` = 'table_name';

-- 查询表中每个列的信息
SELECT `TABLE_SCHEMA`, `TABLE_NAME`, `COLUMN_NAME`, `COLLATION_NAME`
FROM `information_schema`.`COLUMNS`
WHERE `TABLE_NAME` = 'table_name';
```
