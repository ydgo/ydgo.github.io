---
title: "灵丹榜"
date: 2023-05-11T14:39:02+08:00
draft: false
categories: ["Web"]
tags: [Web,斗气大陆]
---
在 Web 开发中，我们需要一些灵丹妙药来提升我们的实力，突破自身桎梏。
<!--more-->

天下丹药，可分九品

## 五品丹药

### OpenAPI 规范

OpenAPI Specification

其实就是一个项目中的满足 OpenAPI 规范的 yaml/json文件，这个文件描述了这个项目所暴露的接口。

基于 Swagger，目前的版本是 OpenAPI 3。

这个东西的作用我觉得有两点：

1. 本地起一个 API 服务，方便在线查看接口文档。
2. 使用代码生成插件，根据 yaml/json文件生成代码。比如 micro 的 [protoc-gen-openapi](https://github.com/google/gnostic/tree/master/cmd/protoc-gen-openapi)

但是也有一个问题：

1. 如何维护这个文档呢？直接编辑 yaml/json文件？
2. 或者说是否支持在线修改、新增接口？


另外，根据代码生成文档不是一个好的点，因为一般是文档设计先于代码设计的。

[API 规范](https://openapi.apifox.cn/)

## 三品丹药

### HTTP 状态码

[状态码注册表](https://www.iana.org/assignments/http-status-codes/http-status-codes.xhtml)



