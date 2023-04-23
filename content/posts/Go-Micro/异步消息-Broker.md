---
title: "异步消息 Broker"
date: 2023-04-23T18:35:47+08:00
draft: false
categories: ["Go-Micro"]
tags: [微服务,Go]
---
本文记录了对 Go 语言下的微服务工具集 Go Micro(v4) 的 Broker 的学习。

代码在我的 [Github](https://github.com/ydgo/rarefire/tree/master/broker) 上。
<!--more-->

## 1. 异步消息

在微服务架构中，众多服务部署在内网中，然后部署一个 frontend 服务对外部各客户端提供服务。frontend 使用 RPC 通信协议调用众多内部服务，
对于同步请求可直接使用 RPC 的方式调用，而有些需求为了实现解耦合，可以通过消息队列的模式来处理，比如邮件发送、消息推送等。所以消息队列是
微服务中比较重要的一个组件。

在 Go Micro 中，通过定义 Broker 接口来提供消息发布和订阅的功能。

## 2. Broker

看一下 Broker 接口的定义

```go
// Broker 接口
type Broker interface {
	Init(...Option) error
	Options() Options
	Address() string
	Connect() error
	Disconnect() error
	Publish(topic string, m *Message, opts ...PublishOption) error
	Subscribe(topic string, h Handler, opts ...SubscribeOption) (Subscriber, error)
	String() string
}

// 消息的处理方法
type Handler func(Event) error

// 发布者发送的消息体
type Message struct {
    Header map[string]string
    Body   []byte
}

// 事件：提供给订阅者程序处理的对象
type Event interface {
    Topic() string
    Message() *Message
    Ack() error
    Error() error
}

// 订阅者，程序订阅一个主题后返回的订阅者
type Subscriber interface {
    Options() SubscribeOptions
    Topic() string
    Unsubscribe() error
}

```

Go Micro 对 Broker 的默认实现是 [Http](https://github.com/go-micro/go-micro/blob/master/broker/http.go)，
也有 [Memory](https://github.com/go-micro/go-micro/blob/master/broker/memory.go) 的实现，然后通过插件形式也实现了很多常用的 Broker：

1. kafka
2. grpc
3. redis
4. nsq
5. ...

可以参考这里：[plugins-brokers](https://github.com/go-micro/plugins/tree/main/v4/broker)

## 3. Broker-Redis

为了更好地了解其实现原理，我便照着插件自己写了一下对 redis 的简单实现。

其中使用的 redis 的库为 [go-redis](github.com/go-redis/redis)，使用了其提供的消息发布订阅的功能。

### 3.1 Broker定义
```go
package broker

type Broker interface {
	Connect() error
	Close() error
	Publish(topic string, m *Message) error
	Subscribe(topic string, h Handler) (Subscriber, error)
}

// Message 消息队列中保存的消息数据
type Message struct {
	Body []byte
}

// Handler 处理消息的方法
type Handler func(Event) error

// Event 将消息发布的整个过程视为一个事件
type Event interface {
	Topic() string
	Message() *Message
}

// Subscriber 订阅者
type Subscriber interface {
	Topic() string
	Unsubscribe() error
}
```

### 3.2 redis 实现

首先根据 Event 接口实现了一个发布者，主要是为了当订阅者处理消息时，如果异常，我们可以向发布者传递什么信息，比如异常信息等。

```go
type publication struct {
	topic   string
	message *Message
	err     error
}


func (p *publication) Topic() string {
	return p.topic
}

func (p *publication) Message() *Message {
	return p.message
}
```

根据订阅者接口实现一个订阅者，主要是循环监听订阅的渠道。
```go

type subscriber struct {
	pubSub *redis.PubSub
	topic  string
	handle Handler
}

func (s *subscriber) recv() {
	defer s.pubSub.Close()
	for msg := range s.pubSub.Channel() {
		var m Message
		m.Body = []byte(msg.Payload)
		p := publication{
			topic:   msg.Channel,
			message: &m,
		}
		if p.err = s.handle(&p); p.err != nil {
			break
		}
		// handle error?
	}
}

// Unsubscribe unsubscribes the subscriber and frees the connection.
func (s *subscriber) Unsubscribe() error {
	return s.pubSub.Unsubscribe(s.topic)
}

// Topic returns the topic of the subscriber.
func (s *subscriber) Topic() string {
	return s.topic
}
```

实现一个具体的 broker，主要是对外提供公共方法，这里我们实现了一个 redis broker。
```go
type redisBroker struct {
	sync.Mutex
	addr      string
	connected bool
	c         *redis.Client
}

func (r *redisBroker) Connect() error {
	r.Lock()
	defer r.Unlock()
	if r.connected {
		return errors.New("already connected")
	}
	r.c = redis.NewClient(&redis.Options{
		Addr:         r.addr,
		MinIdleConns: 5,
		DB:           0,
	})
	return r.c.Ping().Err()
}

func (r *redisBroker) Close() error {
	r.Lock()
	defer r.Unlock()
	r.connected = false
	return r.c.Close()
}

func (r *redisBroker) Publish(topic string, m *Message) error {
	return r.c.Publish(topic, string(m.Body)).Err()
}

func (r *redisBroker) Subscribe(topic string, h Handler) (Subscriber, error) {
	s := subscriber{
		pubSub: r.c.Subscribe(topic),
		topic:  topic,
		handle: h,
	}
	go s.recv()
	return &s, nil
}

func NewRedisBroker(addr string) Broker {
	return &redisBroker{
		addr:      addr,
		connected: false,
	}
}
```

### 3.3 测试

通过测试用例测试代码的正确性和演示一下简单的用法。

```go
package broker

import (
	"fmt"
	"reflect"
	"sort"
	"testing"
	"time"
)

var (
	addr  = "127.0.0.1:6379"
	topic = "rarefire:broker:test1"
)

func TestNewRedisBroker(t *testing.T) {
	broker := NewRedisBroker(addr)
	if err := broker.Connect(); err != nil {
		t.Fatal("connect fail: ", err)
	}
}

func subscribe(t *testing.T, b Broker, topic string, handle Handler) Subscriber {
	s, err := b.Subscribe(topic, handle)
	if err != nil {
		t.Error(err)
	}
	return s
}

func publish(t *testing.T, b Broker, topic string, msg *Message) {
	if err := b.Publish(topic, msg); err != nil {
		t.Error(err)
	}
}

func unsubscribe(t *testing.T, s Subscriber) {
	if err := s.Unsubscribe(); err != nil {
		t.Error(err)
	}
}

func TestBroker(t *testing.T) {
	broker := NewRedisBroker(addr)
	if err := broker.Connect(); err != nil {
		t.Fatal("connect fail: ", err)
	}
	msgs := make(chan string, 10)
	go func() {
		s1 := subscribe(t, broker, topic, func(p Event) error {
			m := p.Message()
			msgs <- fmt.Sprintf("s1:%s", string(m.Body))
			return nil
		})
		s2 := subscribe(t, broker, topic, func(p Event) error {
			m := p.Message()
			msgs <- fmt.Sprintf("s2:%s", string(m.Body))
			return nil
		})

		publish(t, broker, topic, &Message{Body: []byte("hello")})
		publish(t, broker, topic, &Message{Body: []byte("world")})

		unsubscribe(t, s1)
		time.Sleep(time.Second)

		publish(t, broker, topic, &Message{Body: []byte("other")})

		unsubscribe(t, s2)
		time.Sleep(time.Second)

		publish(t, broker, topic, &Message{Body: []byte("none")})

		time.Sleep(time.Second)
		close(msgs)
	}()

	var actual []string
	for msg := range msgs {
		actual = append(actual, msg)
	}
	exp := []string{
		"s1:hello",
		"s2:hello",
		"s1:world",
		"s2:world",
		"s2:other",
	}

	// Order is not guaranteed.
	sort.Strings(actual)
	sort.Strings(exp)
	if !reflect.DeepEqual(actual, exp) {
		t.Fatalf("expected %v, got %v", exp, actual)
	}

}

```

## 4. 总结

1. 接口定义和具体实现分开方便扩展，达到插件化。