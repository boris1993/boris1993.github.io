---
title: "MySQL Workbench中各个列属性的含义"
source: https://www.boris1993.com/column-flags-in-mysql-workbench.html
date: 2019-12-14
updated: 2026-10-10
tags: [MySQL, column flag]
categories: [学知识]
---

# MySQL Workbench中各个列属性的含义

-   `PK`: 主键(Primary Key)
-   `NN`: 非空(Not Null)
-   `UQ`: 唯一索引(Unique Index)
-   `BIN`: 二进制(Binary) 将数据储存为二进制字符串
-   `UN`: 无符号的(Unsigned)
-   `ZF`: 零填充的(Zero Fill) 如：INT(5)的列中，`12`会被填充为`00012`
-   `AI`: 自增长的(Auto Increment)
-   `G`: 生成出来的(Generated) 如：根据公式从其它列中生成的数据

\[^1\]: What do column flags mean in MySQL Workbench?  
\[^2\]: Columns Tab - MySQL Workbench Manual
