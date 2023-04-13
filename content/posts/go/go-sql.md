---
title: "Go database/sql 源码解析"
date: 2023-04-04T17:00:38+08:00
draft: false
categories: ["Go"]
tags: [Go,Sql]
---

本文记录了在 Go 语言中实现数据库 API 的包`database/sql`的源码分析。
<!--more-->

<div class="fa-file-pdf">
    <embed id="pdfPlayer" src="go-sql-pool.pdf" type="application/pdf" width="100%" height="600" >
</div>


[类图生成工具](https://github.com/jfeliu007/goplantuml)

[类图在线生成平台](https://www.dumels.com/)

[Go SQLDrivers](https://github.com/golang/go/wiki/SQLDrivers)

[数据库系统隔离级别](https://en.wikipedia.org/wiki/Isolation_(database_systems)#Isolation_levels)