---
title: "ES 常用 API"
date: 2024-08-24T15:36:21+08:00
draft: false
categories: ["ELK"]
tags: [ES]
---
记录 ES 使用中常见的 API。
<!--more-->

## 1. 运维

~~~text
# 查看 index 的配置
GET /index/_settings

# 查看节点的线程池信息
GET /_nodes/thread_pool

# 查看所有节点当前线程池状态
GET /_cat/thread_pool?v
~~~

## 2. 测试

~~~text
# 创建索引
put /k2es
{
  "mappings": {
      "properties" : {
        "_app" : {
          "type" : "keyword"
        },
        "_datamodel" : {
          "type" : "keyword"
        }
      }
    }
}
~~~