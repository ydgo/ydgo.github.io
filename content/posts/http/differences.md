---
title: "HTTP/1.0 和 HTTP/1.1的不同之处"
date: 2023-05-06T10:34:06+08:00
draft: false
categories: ["HTTP"]
tags: [http]
---

## 1. 可扩展性


在 HTTP/1.1 最终规范版本和 HTTP/1.0 版本之间，有一些非正式的草案和文档，所以 HTTP/1.1 需要兼容这些版本中的内容，即使是错误的。

引入 OPTIONS 方法

增加 Upgrading 请求头来支持协议升级


## 2. 缓存

作用

    1. 消除延迟感知
    2. 减少带宽
    3. 降低服务器或者代理服务器的负载

HTTP/1.0 中的缓存

- 使用 Expires 响应头标记响应的过期时间

- 发送请求时使用 If-Modified-Since 条件请求头，指定缓存资源的响应头中的 Last-Modified 的值，来检查缓存资源当前的有效性。
然后服务器可以发送 304 (Not Modified) 状态码进行响应，暗示缓存的资源未进行修改，或者发送 200 响应来替换缓存的资源。 

有一些缺点导错误缓存一些不应该被缓存的资源，以及未能缓存一些本应该被缓存的资源。


HTTP/1.0 中的缓存

为缓存提供显示和可扩展的协议机制，增强了缓存功能。


## 3. 带宽优化


## 参考

[Key differences between HTTP/1.0 and HTTP/1.1](https://www.ra.ethz.ch/cdstore/www8/data/2136/pdf/pd1.pdf)

