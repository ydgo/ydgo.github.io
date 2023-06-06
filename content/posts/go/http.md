---
title: "net/http 简单使用"
date: 2023-06-06T16:25:06+08:00
draft: false
categories: ["Go"]
tags: [Go,http]
---
介绍 Go 的官方库对 HTTP 客户端和服务端工具的实现。
<!--more-->

官方的库文档很全面，很多方法可以直接看文档。这里只是对 `Handler` 和 `HandleFunc` 的使用说一下自己的理解。

下面先看下代码：

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func Pong(w http.ResponseWriter, r *http.Request) {
	_, _ = fmt.Fprintln(w, "pong")
}

type Index struct {
}

func (i *Index) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	_, _ = fmt.Fprintln(w, "hello world")
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/ping", Pong)
	mux.HandleFunc("/not_found1", http.NotFound)
	mux.Handle("/not_found2", http.NotFoundHandler())
	mux.Handle("/", &Index{})
	mux.Handle("/books", http.RedirectHandler("/ping", http.StatusFound))
	log.Fatal(http.ListenAndServe(":8080", mux))

}

```

输出如下：

```shell
$ curl http://127.0.0.1:8080/ping
pong

$ curl http://127.0.0.1:8080/not_found1
404 page not found

$ curl http://127.0.0.1:8080/not_found2
404 page not found

$ curl http://127.0.0.1:8080/
hello world

$ curl http://127.0.0.1:8080/books
<a href="/ping">Found</a>.

$ curl http://127.0.0.1:8080/books -L
pong
```

看一下这两个东西的源码

## Handler
```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

## HandleFunc

```go
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}

```


我们再看一个具体的Handler

## NotFound
```go
// NotFound replies to the request with an HTTP 404 not found error.
func NotFound(w ResponseWriter, r *Request) { Error(w, "404 page not found", StatusNotFound) }

// NotFoundHandler returns a simple request handler
// that replies to each request with a ``404 page not found'' reply.
func NotFoundHandler() Handler { return HandlerFunc(NotFound) }
```

理解如下：

1. `Handler`是一个接口类型，有一个方法用来处理请求
2. `HandlerFunc`是一个函数类型，且实现了`Handler`接口，所以我们可以把一个满足方法签名的普通函数转换为一个`Handler`
3. `NotFound`是一个满足`Handler`接口下方法的签名的普通函数
4. 所以我们可以通过`HandlerFunc(NotFound)`将 NotFound 转化为一个 Handler，`NotFoundHandler()`方法其实就是这样做的

## 结论
1. `mux.HandlerFunc()`其实就是一种便捷方式，可以让我们通过一个函数去处理请求。
2. `mux.Handle()`可以让我们使用任意自定义对象去处理一个请求，前提是这个对象实现了`Handler`接口。
