---
title: "gRPC(Go)-3-Demo"
date: 2023-04-28T13:53:44+08:00
draft: false
categories: ["gRPC"]
tags: [RPC,Go]
---

本文使用 gRPC 框架写一个 demo。不会使用复杂的功能，就简单的 RPC 调用。
<!--more-->

## 1. 目录结构

```shell
.
├── client
│   └── main.go
├── go.mod
├── go.sum
├── helloworld
│   ├── helloworld_grpc.pb.go
│   ├── helloworld.pb.go
│   └── helloworld.proto
└── server
    └── main.go

```

> 首先我们需要准备开发工具（本次环境如下）
> 1. go v1.20.0
> 2. protoc v3.0.0
> 3. protoc-gen-go v1.28.1
> 4. protoc-gen-go-grpc v1.2.0

## 2. 服务定义

首先定义好服务接口和相关参数。

```protobuf
syntax = "proto3";
option go_package = "./helloworld;helloworld";
package helloworld;

service HelloWorld {
  rpc SayHello (HelloRequest) returns (HelloResponse) {}
}

message HelloRequest {
  string name = 1;
}

message HelloResponse {
  string message = 1;
}
```

然后进行编译，最后会生成一个 pb.go 和一个 grpc.pb.go。

```shell
protoc --go_out=. --go-grpc_out=. ./helloworld/helloworld.proto
```

## 3. 服务端代码

```go
package main

import (
	"context"
	"google.golang.org/grpc"
	pb "grpc_demo/helloworld"
	"log"
	"net"
)

type helloServer struct {
	pb.UnimplementedHelloWorldServer
}

func (h *helloServer) SayHello(ctx context.Context, req *pb.HelloRequest) (*pb.HelloResponse, error) {
	name := req.GetName()
	log.Println("hello server receive: ", name)
	return &pb.HelloResponse{Message: "hello " + name}, nil
}

func main() {
	lis, err := net.Listen("tcp", "127.0.0.1:8080")
	if err != nil {
		log.Fatalln("hello server listen fail: ", err.Error())
	}
	grpcServer := grpc.NewServer()
	hServer := &helloServer{}
	pb.RegisterHelloWorldServer(grpcServer, hServer)
	if err := grpcServer.Serve(lis); err != nil {
		log.Fatalln("hello server serve fail: ", err.Error())
	}

}

```

## 4. 客户端代码

```go
package main

import (
	"context"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	pb "grpc_demo/helloworld"
	"log"
)

func main() {
	conn, err := grpc.Dial("127.0.0.1:8080", grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalln("client dail fail: ", err.Error())
	}
	client := pb.NewHelloWorldClient(conn)

	resp, err := client.SayHello(context.Background(), &pb.HelloRequest{Name: "xiaoyan"})
	if err != nil {
		log.Fatalln("client call fail: ", err.Error())
	}
	log.Println("client call response: ", resp.GetMessage())

}

```

## 5. 测试

首先运行服务端

```shell
cd server
go run .
```

再运行客户端

```go
cd client
go run .
```

最后输出

```shell
2023/04/28 13:50:53 client call response:  hello xiaoyan
```

## 6. 总结

所以总体步骤就是：

1. 定义服务 proto 文件。
2. 编译生成相关代码。
3. 实现服务定义的方法。
4. 开发服务端代码，将自己的服务注册到 grpc 的服务中去，然后开启 grpc 服务。
5. 开发客户端代码，主要是连接，然后调用方法。