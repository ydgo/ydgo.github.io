---
title: "Go 应用完整的配置解决方案-Viper"
date: 2023-05-08T11:03:50+08:00
draft: false
categories: ["Go"]
tags: [Go,Config]
---
本文对 Viper 进行一个简单的学习记录。
<!--more-->

这个项目的[文档](https://pkg.go.dev/github.com/spf13/viper)写得很清楚，直接总结以下特性：

- 多种类型的配置文件
- 多种远程配置来源
- 配置的多种优先级
- 可以从环境变量读取
- 可以设置默认值
- 可以监听配置文件的变化（实时读取配置文件）
- 单例模式/多实例模式
- 可以将配置映射到结构体
- 满足 [12-Factor apps](https://12factor.net/zh_cn/)
- 当前版大小写不敏感


