---
title: "Bind"
date: 2023-04-25T16:39:06+08:00
draft: false
categories: ["Gin"]
tags: [Go,Gin]
---
本文介绍了 Gin 框架中关于数据绑定的基础知识。

<!--more-->

## 1. 什么是 Gin Binding

Gin 绑定用于将 JSON、XML、路径参数、表单数据等序列化为结构体，并自带了验证功能。

Binding 的接口定义为

```go
// 用于绑定请求中存在的数据，例如 JSON 请求正文、查询参数或表单 POST。
type Binding interface {
	Name() string
	Bind(*http.Request, any) error
}

// BindingBody 为 Binding 添加了 BindBody 方法。 BindBody 与 Bind 类似，但它从提供的字节而不是 req.Body 中读取主体。
type BindingBody interface {
    Binding
    BindBody([]byte, any) error
}

// BindingUri 为 Binding 添加了 BindUri 方法。 BindUri 与 Bind 类似，但它读取的是 Params。
type BindingUri interface {
    Name() string
    BindUri(map[string][]string, any) error
}

```

## 2. 绑定的数据格式类型

Gin 实现的几种用于绑定的数据格式

```go
// 它们实现了 Binding 接口，可用于将请求中存在的数据绑定到结构实例。
var (
	JSON          = jsonBinding{}
	XML           = xmlBinding{}
	Form          = formBinding{}
	Query         = queryBinding{}
	FormPost      = formPostBinding{}
	FormMultipart = formMultipartBinding{}
	ProtoBuf      = protobufBinding{}
	MsgPack       = msgpackBinding{}
	YAML          = yamlBinding{}
	Uri           = uriBinding{}
	Header        = headerBinding{}
	TOML          = tomlBinding{}
)
```

Gin 也会根据 HTTP 方法和内容类型返回适当的 Binding 实例。

```go
func Default(method, contentType string) Binding {
	if method == http.MethodGet {
		return Form
	}

	switch contentType {
	case MIMEJSON:
		return JSON
	case MIMEXML, MIMEXML2:
		return XML
	case MIMEPROTOBUF:
		return ProtoBuf
	case MIMEMSGPACK, MIMEMSGPACK2:
		return MsgPack
	case MIMEYAML:
		return YAML
	case MIMETOML:
		return TOML
	case MIMEMultipartPOSTForm:
		return FormMultipart
	default: // case MIMEPOSTForm:
		return Form
	}
}
```

## 3. 几种绑定的区别

Gin提供了两类绑定方法：
- Type - Must bind

   - Methods - Bind, BindJSON, BindXML, BindQuery, BindYAML
   - Behavior - 这些方法属于 MustBindWith 的具体调用。 如果发生绑定错误，则请求终止，并触发 c.AbortWithError(400, err).SetType(ErrorTypeBind)。响应状态码被设置为 400 并且 Content-Type 被设置为 text/plain; charset=utf-8。 如果您在此之后尝试设置响应状态码，Gin会输出日志 [GIN-debug] [WARNING] Headers were already written. Wanted to override status code 400 with 422。 如果您希望更好地控制绑定，考虑使用 ShouldBind 等效方法。
- Type - Should bind
    - Methods - ShouldBind, ShouldBindJSON, ShouldBindXML, ShouldBindQuery, ShouldBindYAML
    - Behavior - 这些方法属于 ShouldBindWith 的具体调用。 如果发生绑定错误，Gin 会返回错误并由开发者处理错误和请求。

先看下是怎么使用的吧。

```go
package main

import (
	"github.com/gin-gonic/gin"
)

// 使用 struct tag 来标记
type album struct {
	ID     string  `json:"id"`
	Title  string  `json:"title"`
	Artist string  `json:"artist"`
	Price  float64 `json:"price"`
}

var albums = []album{
	{ID: "1", Title: "Blue Train", Artist: "John Coltrane", Price: 56.99},
	{ID: "2", Title: "Jeru", Artist: "Gerry Mulligan", Price: 17.99},
	{ID: "3", Title: "Sarah Vaughan and Clifford Brown", Artist: "Sarah Vaughan", Price: 39.99},
}


func postAlbums(c *gin.Context) {
	var newAlbum album

	// Call BindJSON to bind the received JSON to
	// newAlbum.
	// 使用 BindJSON 来绑定
	if err := c.BindJSON(&newAlbum); err != nil {
		return
	}

	// Add the new album to the slice.
	albums = append(albums, newAlbum)
	c.IndentedJSON(http.StatusCreated, newAlbum)
}

func main() {
	router := gin.Default()
	router.POST("/albums", postAlbums)
	router.Run("localhost:8080")
}


```

### 3.1 MustBindWith

绑定失败时，将响应状态代码设置为 400 或在输入无效时中止。

Gin 根据上面这些绑定类型对外提供一些绑定方法供我们使用。

我们可以直接根据我们的需求去使用对应的方法。注意这些方法底层都调用了 `MustBindWith` 这个方法。

但是我们要注意，这种绑定方法会直接向客户端返回失败的响应。

```go
func (c *Context) Bind(obj any) error {
	b := binding.Default(c.Request.Method, c.ContentType())
	return c.MustBindWith(obj, b)
}

// BindJSON is a shortcut for c.MustBindWith(obj, binding.JSON).
func (c *Context) BindJSON(obj any) error {
	return c.MustBindWith(obj, binding.JSON)
}

// BindXML is a shortcut for c.MustBindWith(obj, binding.BindXML).
func (c *Context) BindXML(obj any) error {
	return c.MustBindWith(obj, binding.XML)
}

// BindQuery is a shortcut for c.MustBindWith(obj, binding.Query).
func (c *Context) BindQuery(obj any) error {
	return c.MustBindWith(obj, binding.Query)
}

// BindYAML is a shortcut for c.MustBindWith(obj, binding.YAML).
func (c *Context) BindYAML(obj any) error {
	return c.MustBindWith(obj, binding.YAML)
}

// BindTOML is a shortcut for c.MustBindWith(obj, binding.TOML).
func (c *Context) BindTOML(obj interface{}) error {
	return c.MustBindWith(obj, binding.TOML)
}

// BindHeader is a shortcut for c.MustBindWith(obj, binding.Header).
func (c *Context) BindHeader(obj any) error {
	return c.MustBindWith(obj, binding.Header)
}

// BindUri binds the passed struct pointer using binding.Uri.
// It will abort the request with HTTP 400 if any error occurs.
func (c *Context) BindUri(obj any) error {
	if err := c.ShouldBindUri(obj); err != nil {
		c.AbortWithError(http.StatusBadRequest, err).SetType(ErrorTypeBind) //nolint: errcheck
		return err
	}
	return nil
}
```

当然我们也可以通过将绑定类型当做参数来使用，通过使用以下方法。比如我们想使用自定义的绑定类型，就可以使用这种方式，将自定义的绑定类型当做参数传递进去。

```go
// MustBindWith 使用指定的绑定引擎绑定传递的结构指针。如果发生任何错误，它将使用 HTTP 400 中止请求。
func (c *Context) MustBindWith(obj any, b binding.Binding) error {
	if err := c.ShouldBindWith(obj, b); err != nil {
		c.AbortWithError(http.StatusBadRequest, err).SetType(ErrorTypeBind) //nolint: errcheck
		return err
	}
	return nil
}

```

### 3.2 ShouldBindWith

与 Bind() 类似，但此方法不会将响应状态代码设置为 400 或在输入无效时中止。

具体对外提供的方法如下：

```go
func (c *Context) ShouldBind(obj any) error {
	b := binding.Default(c.Request.Method, c.ContentType())
	return c.ShouldBindWith(obj, b)
}

// ShouldBindJSON is a shortcut for c.ShouldBindWith(obj, binding.JSON).
func (c *Context) ShouldBindJSON(obj any) error {
	return c.ShouldBindWith(obj, binding.JSON)
}

// ShouldBindXML is a shortcut for c.ShouldBindWith(obj, binding.XML).
func (c *Context) ShouldBindXML(obj any) error {
	return c.ShouldBindWith(obj, binding.XML)
}

// ShouldBindQuery is a shortcut for c.ShouldBindWith(obj, binding.Query).
func (c *Context) ShouldBindQuery(obj any) error {
	return c.ShouldBindWith(obj, binding.Query)
}

// ShouldBindYAML is a shortcut for c.ShouldBindWith(obj, binding.YAML).
func (c *Context) ShouldBindYAML(obj any) error {
	return c.ShouldBindWith(obj, binding.YAML)
}

// ShouldBindTOML is a shortcut for c.ShouldBindWith(obj, binding.TOML).
func (c *Context) ShouldBindTOML(obj interface{}) error {
	return c.ShouldBindWith(obj, binding.TOML)
}

// ShouldBindHeader is a shortcut for c.ShouldBindWith(obj, binding.Header).
func (c *Context) ShouldBindHeader(obj any) error {
	return c.ShouldBindWith(obj, binding.Header)
}

// ShouldBindUri binds the passed struct pointer using the specified binding engine.
func (c *Context) ShouldBindUri(obj any) error {
	m := make(map[string][]string)
	for _, v := range c.Params {
		m[v.Key] = []string{v.Value}
	}
	return binding.Uri.BindUri(m, obj)
}

// ShouldBindWith binds the passed struct pointer using the specified binding engine.
// See the binding package.
func (c *Context) ShouldBindWith(obj any, b binding.Binding) error {
	return b.Bind(c.Request, obj)
}
```


### 3.3 ShouldBindBodyWith

ShouldBindBodyWith 与 ShouldBindWith 类似，但它将请求体存储到上下文中，并在再次调用时重用。注意：此方法在绑定之前读取正文。所以如果你只需要调用一次，你应该使用 ShouldBindWith 以获得更好的性能。

```go
func (c *Context) ShouldBindBodyWith(obj any, bb binding.BindingBody) (err error) {
	var body []byte
	if cb, ok := c.Get(BodyBytesKey); ok {
		if cbb, ok := cb.([]byte); ok {
			body = cbb
		}
	}
	if body == nil {
		body, err = io.ReadAll(c.Request.Body)
		if err != nil {
			return err
		}
		c.Set(BodyBytesKey, body)
	}
	return bb.BindBody(body, obj)
}
```

所以，除了 `ShouldBindBodyWith` 外，其他的绑定方法都是从 ctx 的缓冲区去读数据，读了一次后，数据就没了，所以只能绑定一次。


## 4. 自带的数据验证

Gin 在内部使用 [validator](https://github.com/go-playground/validator/tree/v10.6.1) 包进行验证。这个包验证器提供了一组广泛的内置验证，包括必需的、类型验证和字符串验证。

### 4.1 示例
```go

type URI struct {
   Details string `json:"name" uri:"details" binding:"required"`
}


// Validating phone numbers, emails, and country codes
/*
   {
      "firstName": "John",
      "lastName": "Mark",
      "email": "jmark@example.com",
      "phone": "+11234567890",
      "countryCode": "US"
   }
*/
type Body struct {
    FirstName string `json:"firstName" binding:"required"`
    LastName string `json:"lastName" binding:"required"`
    Email string `json:"email" binding:"required,email"`
    Phone string `json:"phone" binding:"required,e164"`
    CountryCode string `json:"countryCode" binding:"required,iso3166_1_alpha2"`
}

type Body struct {
    StartDate time.Time `form:"start_date" binding:"required,ltefield=EndDate" time_format:"2006-01-02"`
    EndDate time.Time `form:"end_date" binding:"required" time_format:"2006-01-02"`
}
```

常见约束可参考这里：https://pkg.go.dev/github.com/go-playground/validator/v10


### 4.2 处理验证错误

在前面的示例中，我们使用 AbortWithError 函数将 HTTP 错误代码发送回客户端，但我们没有发送有意义的错误消息。因此，我们可以通过将有意义的验证错误消息作为 JSON 输出发送来改进端点：

```go
package main

import (
	"errors"
	"github.com/gin-gonic/gin"
	"github.com/go-playground/validator/v10"
	"net/http"
)

type Body struct {
	Product string `form:"product" binding:"required,email,len=5"`
}

type ErrorMsg struct {
	Field   string      `json:"field"`
	Message string      `json:"message"`
	Value   interface{} `json:"value"`
}

func Index(ctx *gin.Context) {
	body := Body{}
	if err := ctx.ShouldBind(&body); err != nil {
		var ve validator.ValidationErrors
		if errors.As(err, &ve) {
			out := make([]ErrorMsg, len(ve))
			for i, fe := range ve {
				out[i] = ErrorMsg{fe.Field(), fe.Error(), fe.Value()}
			}
			ctx.AbortWithStatusJSON(http.StatusBadRequest, gin.H{"errors": out})
		}
		return
	}
	ctx.JSON(http.StatusAccepted, &body)
}

func main() {
	engine := gin.New()
	engine.GET("", Index)
	engine.Run(":8080")
}

```

请求一下

```shell
curl http://localhost:8080?product=dsdas@gmail.com

```

返回数据如下

```json
{
  "errors": [
    {
      "field": "Product",
      "message": "Key: 'Body.Product' Error:Field validation for 'Product' failed on the 'len' tag",
      "value": "dsdas@gmail.com"
    }
  ]
}
```

## 5. 自定义绑定格式类型

比如我们想要获取请求的查询参数，一般会在结构体字段上加上 tag: form，我们想换成 json，怎么办呢？

```go
// Copyright 2014 Manu Martinez-Almeida. All rights reserved.
// Use of this source code is governed by a MIT style
// license that can be found in the LICENSE file.

package mybind

import (
	"errors"
	"github.com/gin-gonic/gin/binding"
	"net/http"
)

const defaultMemory = 32 << 20

type formBinding struct{}

func (formBinding) Name() string {
	return "form"
}

func NewBinding() binding.Binding {
	return formBinding{}
}

func (formBinding) Bind(req *http.Request, obj any) error {
	if err := req.ParseForm(); err != nil {
		return err
	}
	if err := req.ParseMultipartForm(defaultMemory); err != nil && !errors.Is(err, http.ErrNotMultipart) {
		return err
	}
	// 将原本这里的 mapForm 换成 MapFormWithTag 即可
	if err := binding.MapFormWithTag(obj, req.Form, "json"); err != nil {
		return err
	}
	return validate(obj)
}

func validate(obj interface{}) error {
	if binding.Validator == nil {
		return nil
	}
	return binding.Validator.ValidateStruct(obj)
}

```

那我们就可以这样使用

```go


type Body struct {
    Name string `json:"name" binding:"required,palindrome"` // 这里使用json替换form
}

func checkPalindrome(ctx *gin.Context) {
	body := Body{}
	// 这里使用我们自定义的NewBinding
	if err := ctx.ShouldBindWith(&body, mybind.NewBinding()); err != nil {
		var ve validator.ValidationErrors
		if errors.As(err, &ve) {
			out := make([]ErrorMsg, len(ve))
			for i, fe := range ve {
				out[i] = ErrorMsg{fe.Field(), fe.Error(), fe.Value()}
			}
			ctx.AbortWithStatusJSON(http.StatusBadRequest, gin.H{"errors": out})
		}
		return
	}
	ctx.JSON(http.StatusAccepted, &body)
}

```

## 6. 自定义数据验证格式

比如我们写一个检查字符串是否是回文格式的验证。

```go
package main

import (
	"errors"
	"gin_demo/mybind"
	"github.com/gin-gonic/gin"
	"github.com/gin-gonic/gin/binding"
	"github.com/go-playground/validator/v10"
	"net/http"
	"reflect"
)

type Body struct {
	Name string `json:"name" binding:"required,palindrome"`
}

type ErrorMsg struct {
	Field   string      `json:"field"`
	Message string      `json:"message"`
	Value   interface{} `json:"value"`
}

func reverse(s []byte) []byte {
	newS := make([]byte, len(s))
	for i, j := 0, len(s)-1; i <= j; i, j = i+1, j-1 {
		newS[i], newS[j] = s[j], s[i]
	}
	return newS
}

// 检查字符串是否是回文形式，如果是返回 true
var palindrome validator.Func = func(fl validator.FieldLevel) bool {
	word, ok := fl.Field().Interface().(string)
	if ok {
		return reflect.DeepEqual([]byte(word), reverse([]byte(word)))
	}
	return true
}

func checkPalindrome(ctx *gin.Context) {
	body := Body{}
	if err := ctx.ShouldBindWith(&body, mybind.NewBinding()); err != nil {
		var ve validator.ValidationErrors
		if errors.As(err, &ve) {
			out := make([]ErrorMsg, len(ve))
			for i, fe := range ve {
				out[i] = ErrorMsg{fe.Field(), fe.Error(), fe.Value()}
			}
			ctx.AbortWithStatusJSON(http.StatusBadRequest, gin.H{"errors": out})
		}
		return
	}
	ctx.JSON(http.StatusAccepted, &body)
}

func main() {
	engine := gin.New()
	// 注册我们的验证器
	if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
		_ = v.RegisterValidation("palindrome", palindrome)
	}
	engine.GET("", checkPalindrome)
	engine.Run(":8080")
}

```

请求一下
```shell
curl http://127.0.0.1:8080/?name=level
```

返回
```json
{
"name": "level"
}
```

请求一下
```shell
curl http://127.0.0.1:8080/?name=123
```

返回
```json
{
    "errors": [
        {
        "field": "Name",
        "message": "Key: 'Body.Name' Error:Field validation for 'Name' failed on the 'palindrome' tag",
        "value": "123"
        }
    ]
}
```

参考

https://blog.logrocket.com/gin-binding-in-go-a-tutorial-with-examples/