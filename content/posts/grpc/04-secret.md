---
title: "gRPC(Go)-5-通过SSL/TLS建立安全连接"
date: 2023-04-28T16:10:41+08:00
draft: false
categories: ["gRPC"]
tags: [RPC,Go]
---
本文记录了gRPC 中如何通过 TLS 证书建立安全连接，让数据能够加密处理，包括证书制作和CA签名校验等。
<!--more-->

## 1. 概述

gRPC 提供了内置的客户端和服务端之间的身份验证和数据加密功能。我们也可以定义自己的身份验证系统。

- **SSL/TLS**
    > 通过证书进行数据加密。
- **ALTS**
    > Google开发的一种双向身份验证和传输加密系统。
- **Token-based authentication with Google**

## 2. With server authentication SSL/TLS

### 2.1 Client 
```go
creds, _ := credentials.NewClientTLSFromFile(certFile, "")
conn, _ := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(creds))
// error handling omitted
client := pb.NewGreeterClient(conn)
// ...
```

### 2.2 Server
```go
creds, _ := credentials.NewServerTLSFromFile(certFile, keyFile)
s := grpc.NewServer(grpc.Creds(creds))
lis, _ := net.Listen("tcp", "localhost:50051")
// error handling omitted
s.Serve(lis)
```

## 3. server-side TLS

服务端 TLS 具体包含以下几个步骤：

1. 制作证书，包含服务端证书和 CA 证书；
2. 服务端启动时加载证书；
3. 客户端连接时使用CA 证书校验服务端证书有效性。

### 3.1 证书制作

首先在项目目录下新建 x509 目录，在此目录下执行以下命令生成证书相关文件。

>** CA 证书**

```shell
# 生成key 私钥文件
openssl genrsa -out ca.key 2048

# 生成.csr 证书签名请求文件
openssl req -new -key ca.key -out ca.csr -subj "/C=CN/L=China/O=yd/CN=www.yudong.com"

# 自签名生成.crt 证书文件
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt -subj "/C=CN/L=China/O=yd/CN=www.yudong.com"

# 最后一条将会输出： subject=C = CN, ST = Guangzhou, L = Guangzhou, O = yudong, OU = yudong, CN = www.yudong.com

```

> **服务端证书**

```shell
# 生成key 私钥文件
openssl genrsa -out server.key 2048

# 生成.csr 证书签名请求文件
openssl req -new -key server.key -out server.csr -reqexts SAN -config <(cat /etc/ssl/openssl.cnf <(printf "\n[SAN]\nsubjectAltName=DNS:*.yudong.com,DNS:*.refersmoon.com" -subj "/C=CN/L=China/O=yd/CN=www.yudong.com"

# 签名生成.crt 证书文件
openssl x509 -req -days 3650 -in server.csr -out server.crt -CA ca.crt -CAkey ca.key -CAcreateserial -extensions SAN -extfile  <(cat /etc/ssl/openssl.cnf <(printf "\n[SAN]\nsubjectAltName=DNS:*.yudong.com,DNS:*.refersmoon.com"))


```

到此会生成以下6个文件：
```shell
.
├── ca.crt
├── ca.csr
├── ca.key
├── ca.srl
├── server.crt
├── server.csr
└── server.key
```

会用到的有下面这3个：

1）ca.crt
2）server.key
3）server.crt

### 3.2 服务端代码
```go
func main() {
	lis, err := net.Listen("tcp", "127.0.0.1:8080")
	if err != nil {
		log.Fatalln("hello server listen fail: ", err.Error())
	}
	creds, err := credentials.NewServerTLSFromFile("../x509/server.crt", "../x509/server.key")
	if err != nil {
		log.Fatalf("hello server failed to create credentials: %v", err)
	}

	grpcServer := grpc.NewServer(grpc.Creds(creds))
	hServer := &helloServer{}
	pb.RegisterHelloWorldServer(grpcServer, hServer)
	if err := grpcServer.Serve(lis); err != nil {
		log.Fatalln("hello server serve fail: ", err.Error())
	}

}
```

### 3.3 客户端代码
```go
func main() {
	creds, err := credentials.NewClientTLSFromFile("../x509/ca.crt", "www.yudong.com")
	if err != nil {
		log.Fatalln("client failed to load credentials: ", err.Error())
	}

	conn, err := grpc.Dial("127.0.0.1:8080", grpc.WithTransportCredentials(creds))
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


### 3.4 测试
```shell
# 服务端日志
2023/04/28 18:11:15 hello server receive stream:  xiaoyan:0
2023/04/28 18:11:15 hello server receive stream:  xiaoyan:1
2023/04/28 18:11:15 hello server send stream:  hello xiaoyan:0
2023/04/28 18:11:15 hello server send stream:  hello xiaoyan:1
2023/04/28 18:11:15 hello server receive stream:  xiaoyan:2
2023/04/28 18:11:15 hello server receive stream:  xiaoyan:3
2023/04/28 18:11:15 hello server send stream:  hello xiaoyan:2
2023/04/28 18:11:15 hello server send stream:  hello xiaoyan:3
2023/04/28 18:11:15 hello server receive stream:  xiaoyan:4
2023/04/28 18:11:15 hello server send stream:  hello xiaoyan:4

# 客户端日志
2023/04/28 18:11:15 client send stream:  xiaoyan:0
2023/04/28 18:11:15 client send stream:  xiaoyan:1
2023/04/28 18:11:15 client send stream:  xiaoyan:2
2023/04/28 18:11:15 client send stream:  xiaoyan:3
2023/04/28 18:11:15 client send stream:  xiaoyan:4
2023/04/28 18:11:15 client call response:  hello xiaoyan:0
2023/04/28 18:11:15 client call response:  hello xiaoyan:1
2023/04/28 18:11:15 client call response:  hello xiaoyan:2
2023/04/28 18:11:15 client call response:  hello xiaoyan:3
2023/04/28 18:11:15 client call response:  hello xiaoyan:4

```

## 4. mutual TLS

## 参考

https://grpc.io/docs/guides/auth/#go

https://www.lixueduan.com/posts/grpc/04-encryption-tls/