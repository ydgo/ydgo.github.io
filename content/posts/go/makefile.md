---
title: "Makefile 的基本使用"
date: 2023-05-11T14:00:06+08:00
draft: false
categories: ["Go"]
tags: [Go]
---
记录 Makefile 的在项目中的基本使用方法。
<!--more-->

创建 Makefile 在大型项目中非常有用，它直观地说明了项目的编译过程和方法。

先看一个示例，这个示例是使用 Go Micro 的命令工具创建一个服务时生成的，很标准，很有参考价值。
```makefile

GOPATH:=$(shell go env GOPATH)
.PHONY: init
init:
	go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
	go install github.com/micro/micro/v3/cmd/protoc-gen-micro@latest
	go install github.com/micro/micro/v3/cmd/protoc-gen-openapi@latest

.PHONY: api
api:
	protoc --openapi_out=. --proto_path=. proto/helloworld.proto

.PHONY: proto
proto:
	protoc --proto_path=. --micro_out=. --go_out=:. proto/helloworld.proto
	
.PHONY: build
build:
	go build -o helloworld *.go

.PHONY: test
test:
	go test -v ./... -cover

.PHONY: docker
docker:
	docker build . -t helloworld:latest
```


再看一个示例，这个示例加入了 help 命令，打印出了每个命令的功能。

其中加入`@`符号，可以在命令执行时，不打印出命令本身。
```makefile
APP=blog

.PHONY: help all build windows linux darwin

help:
        @echo "usage: make <option>"
        @echo "options and effects:"
        @echo "    help   : Show help"
        @echo "    all    : Build multiple binary of this project"
        @echo "    build  : Build the binary of this project for current platform"
        @echo "    windows: Build the windows binary of this project"
        @echo "    linux  : Build the linux binary of this project"
        @echo "    darwin : Build the darwin binary of this project"
all:build windows linux darwin
build:
        @go build -o ${APP}
windows:
        @GOOS=windows go build -o ${APP}-windows
linux:
        @GOOS=linux go build -o ${APP}-linux
darwin:
        @GOOS=darwin go build -o ${APP}-darwin
```

对于使用 Go 语言开发的项目，使用 Makefile 是非常舒服的，包括依赖的安装、测试用例的执行、代码编译、docker 打包都很方便。


参考

https://www.51cto.com/article/709065.html