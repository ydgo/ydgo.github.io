---
title: "Go开发规范"
date: 2023-04-10T14:50:05+08:00
draft: false
categories: ["Go"]
tags: [Go,开发规范]
---
Go 语言开发过程中需要注意的开发规范。
<!--more-->

for 循环 + switch break

```go
package main

import "fmt"

func main() {
loop:
	for n := 0; n < 10; n++ {
		switch n {
		case 5:
			break loop
		case 3, 2:
			break
		default:
			fmt.Println("n: ", n)
		}
	}
}

/* output
n:  0
n:  1
n:  4
*/
```

[Effective Go](https://go.dev/doc/effective_go)

[Uber Go 语言编码规范](https://github.com/xxjwxc/uber_go_guide_cn)

[不要只检查错误，优雅地处理他们](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)

[Go 包命名规则](https://blog.golang.org/package-names)

[Go 包样式指南](https://rakyll.org/style-packages/)

[代码格式化工具](https://github.com/xxjwxc/uber_go_guide_cn#linting)

[Functional options for friendly APIs](https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis)

[golangci-lint](https://github.com/golangci/golangci-lint)

[实用的 Go 集合](https://dave.cheney.net/practical-go)

