---
title: "Mysql Ddl"
date: 2023-04-13T18:33:37+08:00
draft: false
categories: ["MySQL"]
tags: [Sql]
---
本文介绍了 MySQL 的 DDL 相关的基础知识。

<!--more-->

# 5.7 Online DDL

https://dev.mysql.com/doc/refman/5.7/en/innodb-online-ddl.html

特性：就地改表、并发DML
好处：

1. 提高响应能力和可用性
2. Lock 子句可以调节DDL的性能和DML并发之间的平衡
3. 更少的磁盘使用空间和I/O开销

子句：
ALGORITHM: 主要用于比较性能
- INPLACE: 通过创建临时文件，将数据读取到临时文件，最后替换数据文件实现表切换
- COPY

LOCK
  - NONE
  - SHARED
  - DEFAULT
  - EXCLUSIVE

## 具体支持的DDL


# 8.0
https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl.html


