---
title: "gRPC(Go)-2-ProtoBuf"
date: 2023-04-27T11:49:36+08:00
draft: false
categories: ["gRPC"]
tags: [RPC,Go]
---

本文主要记录了 Protobuf 的基本使用。包括 编译器 protoc 、Go Plugins 安装及 .proto文件定义、编译等。
<!--more-->

## 1. 概述

**Protocol buffers** 是一种语言无关、平台无关的可扩展机制或者说是数据交换格式，用于序列化结构化数据。
与 XML、JSON 相比，Protocol buffers 序列化后的码流更小、速度更快、操作更简单。

> Protocol Buffers (a.k.a., protobuf) are Google's language-neutral, platform-neutral, extensible mechanism for 
> serializing structured data. You can learn more about it in protobuf's documentation.

## 2. Protocol Compiler

protoc 用于编译 protobuf (.proto文件) 和 protobuf 运行时。使用 C++ 编写的，可以在 [GitHub release page](https://github.com/protocolbuffers/protobuf/releases) 下载。

## 3. Protobuf Runtime

Protobuf 支持多种不同的编程语言，所以针对不同的编程语言，会有不同的实现。
至于这两者的关系，不是很理解，反正是相辅相成的吧。

我知道的有一个优点就是，我们可以写一份 .proto 文件，然后根据不同语言的插件生成对应的代码文件，比如 `.pu.go` 或者 `.pb.py` 等。


## 3. Go Plugins

> protoc-gen-go 工具是 protoc 的编译器插件，protocol buffer 编译器。它增强了 protoc 编译器，使其知道如何为给定的 .proto 文件生成 Go 特定代码。

```shell
go get google.golang.org/protobuf/cmd/protoc-gen-go
```

## 4. Demo

### 4.1 创建 .proto 文件

```protobuf

//声明proto的版本 只有 proto3 才支持 gRPC
syntax = "proto3";
// 将编译后文件输出在 github.com/lixd/grpc-go-example/helloworld/helloworld 目录
option go_package = "github.com/lixd/grpc-go-example/helloworld/helloworld";
// 指定当前proto文件属于helloworld包
package helloworld;

// 定义一个名叫 greeting 的服务
service Greeter {
  // 该服务包含一个 SayHello 方法 HelloRequest、HelloReply分别为该方法的输入与输出
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}
// 具体的参数定义
message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}
```

### 4.2 编译

```shell
protoc --proto_path=IMPORT_PATH  --go_out=OUT_DIR  --go_opt=paths=source_relative path/to/file.proto
```

这里简单介绍一下 golang 的编译姿势:

- proto_path或者-I ：指定 import 路径，可以指定多个参数，编译时按顺序查找，不指定时默认查找当前目录。

    - .proto 文件中也可以引入其他 .proto 文件，这里主要用于指定被引入文件的位置。

- go_out：golang编译支持，指定输出文件路径，其他语言则替换即可，比如 --java_out 等等。

- go_opt：指定参数，比如--go_opt=paths=source_relative就是表明生成文件输出使用相对路径。 path/to/file.proto ：被编译的 .proto 文件放在最后面

```shell
protoc --go_out=. hello_word.proto
```

编译后会生成一个hello_word.pb.go文件。

到此为止就ok了。

参考

https://www.lixueduan.com/posts/grpc/01-protobuf

https://www.lixueduan.com/posts/protobuf/01-import/