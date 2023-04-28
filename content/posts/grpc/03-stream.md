---
title: "gRPC(Go)-4-Stream"
date: 2023-04-28T14:29:23+08:00
draft: false
categories: ["gRPC"]
tags: [RPC,Go]
---

本文在 demo 的基础上演示下流的基本使用。
<!--more-->


## 1. 请求类型

1. UnaryAPI：普通一元方法
2. ServerStreaming：服务端推送流
3. ClientStreaming：客户端推送流
4. BidirectionalStreaming：双向推送流

## 2. 服务定义

```protobuf
syntax = "proto3";
option go_package = "./helloworld;helloworld";
package helloworld;

service HelloWorld {
  rpc SayHello (HelloRequest) returns (HelloResponse) {}                                // 一元请求
  rpc ServerStreamHello (HelloRequest) returns (stream HelloResponse) {}                // 服务端流
  rpc ClientStreamHello (stream HelloRequest) returns (HelloResponse) {}                // 客户端流
  rpc BidirectionalStreamHello (stream HelloRequest) returns (stream HelloResponse) {}  // 双向流
}

message HelloRequest {
  string name = 1;
}

message HelloResponse {
  string message = 1;
}
```

## 3. 服务端流

### 3.1 服务端代码

```go
func (h *helloServer) ServerStreamHello(req *pb.HelloRequest, stream pb.HelloWorld_ServerStreamHelloServer) error {
	name := req.GetName()
	log.Println("hello server receive: ", name)
	for i := 0; i < 5; i++ {
		err := stream.Send(&pb.HelloResponse{Message: "hello " + name + fmt.Sprintf(":%d", i)})
		if err != nil {
			log.Println("hello server send stream fail: ", err.Error())
			return err
		}
	}
	return nil
}
```

### 3.2 客户端代码

```go
func main() {
	conn, err := grpc.Dial("127.0.0.1:8080", grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalln("client dail fail: ", err.Error())
	}
	client := pb.NewHelloWorldClient(conn)

	stream, err := client.ServerStreamHello(context.Background(), &pb.HelloRequest{Name: "xiaoyan"})
	if err != nil {
		log.Fatalln("client call fail: ", err.Error())
	}
	for {
		res, err := stream.Recv()
		if err == io.EOF {
			break
		}
		if err != nil {
			log.Println("client receive stream fail: ", err.Error())
			return
		}
		log.Println("client call response: ", res.GetMessage())
	}
}
```

### 3.3 结果

因为是服务端流，所以客户端一个请求会收到多个响应。
```shell
2023/04/28 14:43:14 client call response:  hello xiaoyan:0
2023/04/28 14:43:14 client call response:  hello xiaoyan:1
2023/04/28 14:43:14 client call response:  hello xiaoyan:2
2023/04/28 14:43:14 client call response:  hello xiaoyan:3
2023/04/28 14:43:14 client call response:  hello xiaoyan:4
```

## 4. 客户端流

### 4.1 服务端代码

```go
func (h *helloServer) ClientStreamHello(stream pb.HelloWorld_ClientStreamHelloServer) error {
	for {
		req, err := stream.Recv()
		if err == io.EOF {
			return stream.SendAndClose(&pb.HelloResponse{Message: "ok"})
		}
		if err != nil {
			log.Println("hello server receive stream fail: ", err.Error())
			break
		}
		log.Println("hello server receive stream: ", req.GetName())
	}
	return nil
}
```

### 4.2 客户端代码

```go
func main() {
	conn, err := grpc.Dial("127.0.0.1:8080", grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalln("client dail fail: ", err.Error())
	}
	client := pb.NewHelloWorldClient(conn)

	stream, err := client.ClientStreamHello(context.Background())
	for i := 0; i < 5; i++ {
		err := stream.Send(&pb.HelloRequest{Name: fmt.Sprintf("xiaoyan:%d", i)})
		if err != nil {
			log.Println("client send stream fail: ", err.Error())
			continue
		}
	}
	resp, err := stream.CloseAndRecv()
	if err != nil {
		log.Println("client receive stream fail: ", err.Error())
		return
	}
	log.Println("client call response: ", resp.GetMessage())
}
```

### 4.3 结果

因为是客户端流，所以客户端多个请求会收到一个响应。
```shell
# 服务端日志
2023/04/28 15:01:24 hello server receive stream:  xiaoyan:0
2023/04/28 15:01:24 hello server receive stream:  xiaoyan:1
2023/04/28 15:01:24 hello server receive stream:  xiaoyan:2
2023/04/28 15:01:24 hello server receive stream:  xiaoyan:3
2023/04/28 15:01:24 hello server receive stream:  xiaoyan:4

# 客户端日志
2023/04/28 15:01:24 client call response:  ok

```

## 5. 双向流

### 5.1 服务端代码

```go
func (h *helloServer) BidirectionalStreamHello(stream pb.HelloWorld_BidirectionalStreamHelloServer) error {
	var (
		wg    sync.WaitGroup
		msgCh = make(chan string)
	)
	wg.Add(1)
	go func() {
		defer wg.Done()
		for v := range msgCh {
			err := stream.Send(&pb.HelloResponse{Message: "hello " + v})
			if err != nil {
				log.Println("hello server send stream fail: ", err.Error())
				continue
			}
			log.Println("hello server send stream: ", "hello "+v)
		}
	}()

	wg.Add(1)
	go func() {
		defer wg.Done()
		for {
			req, err := stream.Recv()
			if err == io.EOF {
				break
			}
			if err != nil {
				log.Println("hello server receive stream fail: ", err.Error())
				continue
			}
			log.Println("hello server receive stream: ", req.GetName())
			msgCh <- req.GetName()
		}
		close(msgCh)
	}()
	wg.Wait()
	return nil
}
```

### 5.2 客户端代码

```go
func main() {
	conn, err := grpc.Dial("127.0.0.1:8080", grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalln("client dail fail: ", err.Error())
	}
	client := pb.NewHelloWorldClient(conn)

	stream, err := client.BidirectionalStreamHello(context.Background())
	if err != nil {
		log.Println("client send and receive stream fail: ", err.Error())
		return
	}
	var wg sync.WaitGroup

	wg.Add(1)
	go func() {
		defer wg.Done()
		for {
			res, err := stream.Recv()
			if err == io.EOF {
				break
			}
			if err != nil {
				log.Println("client receive stream fail: ", err.Error())
				return
			}
			log.Println("client call response: ", res.GetMessage())
		}
	}()
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < 5; i++ {
			msg := fmt.Sprintf("xiaoyan:%d", i)
			err := stream.Send(&pb.HelloRequest{Name: msg})
			if err != nil {
				log.Println("client send stream fail: ", err.Error())
				continue
			}
			log.Println("client send stream: ", msg)
		}
		err = stream.CloseSend()
		if err != nil {
			log.Println("client close send fail: ", err.Error())
		}
	}()

	wg.Wait()
}
```

### 5.3 结果

因为是双向流，所以客户端多个请求会收到多个响应。
```shell
# 服务端日志
2023/04/28 15:39:52 hello server receive stream:  xiaoyan:0
2023/04/28 15:39:52 hello server receive stream:  xiaoyan:1
2023/04/28 15:39:52 hello server send stream:  hello xiaoyan:0
2023/04/28 15:39:52 hello server send stream:  hello xiaoyan:1
2023/04/28 15:39:52 hello server receive stream:  xiaoyan:2
2023/04/28 15:39:52 hello server receive stream:  xiaoyan:3
2023/04/28 15:39:52 hello server send stream:  hello xiaoyan:2

# 客户端日志
2023/04/28 15:39:52 client send stream:  xiaoyan:0
2023/04/28 15:39:52 client send stream:  xiaoyan:1
2023/04/28 15:39:52 client send stream:  xiaoyan:2
2023/04/28 15:39:52 client send stream:  xiaoyan:3
2023/04/28 15:39:52 client send stream:  xiaoyan:4
2023/04/28 15:39:52 client call response:  hello xiaoyan:0
2023/04/28 15:39:52 client call response:  hello xiaoyan:1
2023/04/28 15:39:52 client call response:  hello xiaoyan:2
2023/04/28 15:39:52 client call response:  hello xiaoyan:3
2023/04/28 15:39:52 client call response:  hello xiaoyan:4

```

## 6. 小结

客户端或者服务端都有对应的 推送或者 接收对象，我们只要 不断循环 Recv(),或者 Send() 就能接收或者推送了！

> gRPC Stream 和 goroutine 配合简直完美。通过 Stream 我们可以更加灵活的实现自己的业务。如 订阅，大数据传输等。


1. ServerStream

    - 服务端处理完成后return nil代表响应完成
    - 客户端通过 err == io.EOF判断服务端是否响应完成

2. ClientStream

   - 客户端发送完毕通过`CloseAndRecv关闭stream 并接收服务端响应
   - 服务端通过 err == io.EOF判断客户端是否发送完毕，完毕后使用SendAndClose关闭 stream并返回响应。

3. BidirectionalStream

   - 客户端服务端都通过stream向对方推送数据
   - 客户端推送完成后通过CloseSend关闭流，通过err == io.EOF`判断服务端是否响应完成
   - 服务端通过err == io.EOF判断客户端是否发送完成,通过return nil表示已经完成响应，通过err == io.EOF来判定是否把对方推送的数据全部获取到了。

客户端通过CloseAndRecv或者CloseSend关闭 Stream，服务端则通过SendAndClose或者直接 return nil来返回响应。

参考

https://www.lixueduan.com/posts/grpc/03-stream