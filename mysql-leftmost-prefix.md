---
title: "MySQL的最左匹配原则"
source: https://www.boris1993.com/mysql-leftmost-prefix.html
date: 2023-04-17
updated: 2026-10-10
tags: [MySQL, 面试, 最左匹配原则, leftmost prefix]
categories: [学知识]
---

# MySQL的最左匹配原则

依旧是面试查漏补缺。这次是MySQL的最左匹配原则。

废话不多说，直接看MySQL官方文档给出的例子。

有这么一个表：

```sql
CREATE TABLE test (
    id         INT NOT NULL,
    last_name  CHAR(30) NOT NULL,
    first_name CHAR(30) NOT NULL,
    PRIMARY KEY (id),
    INDEX name (last_name, first_name)
);
```

可以看出，表里面有一个联合索引`(last_name, first_name)`。那么，查询的时候，这些语句就可以成功走索引：

```sql
SELECT * FROM test WHERE last_name = 'Doe';
SELECT * FROM test WHERE last_name = 'Doe' AND first_name = 'John';
SELECT * FROM test WHERE last_name = 'Doe' AND (first_name = 'John' OR first_name = 'Jane');
SELECT * FROM test WHERE last_name = 'Doe' AND first_name >= 'M' AND first_name < 'N';
```

而这些查询就无法使用这个索引：

```sql
SELECT * FROM test WHERE first_name = 'John';
SELECT * FROM test WHERE last_name = 'Doe' OR first_name = 'John';
```

也就是说，上面例子中的`last_name`列就是索引的最左前缀，如果要在查询中使用这个索引，那么条件中必须包含`last_name`，或者同时包含`last_name`和`first_name`。

> MySQL can use multiple-column indexes for queries that test all the columns in the index, or queries that test just the first column, the first two columns, the first three columns, and so on. If you specify the columns in the right order in the index definition, a single composite index can speed up several kinds of queries on the same table.
