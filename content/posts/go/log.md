---
title: "logger的封装使用"
date: 2023-09-14T09:19:26+08:00
draft: true
categories: ["Go"]
tags: [Go,log]
---
如何一步一步封装一个logger
<!--more-->

装的好处是，随时可以更新/切换底层的包，因为底层的包更新更频繁，而业务使用封装的包，稳定性更强

## log 的工作流程
1. 将参数传递给logger
2. 根据格式化函数，格式化打印的信息
3. 将格式化后的字符串输出到某个地方，比如 stdout、file、remote

以 Go 的 log 为例
~~~go
func (l *Logger) Output(calldepth int, s string) error {
	now := time.Now() // get this early.
	var file string
	var line int
	l.mu.Lock()
	defer l.mu.Unlock()
	if l.flag&(Lshortfile|Llongfile) != 0 {
		// Release lock while getting caller info - it's expensive.
		l.mu.Unlock()
		var ok bool
		_, file, line, ok = runtime.Caller(calldepth)
		if !ok {
			file = "???"
			line = 0
		}
		l.mu.Lock()
	}
	l.buf = l.buf[:0]
	// 根据格式化函数，格式化打印的信息
	l.formatHeader(&l.buf, now, file, line)
	l.buf = append(l.buf, s...)
	if len(s) == 0 || s[len(s)-1] != '\n' {
		l.buf = append(l.buf, '\n')
	}
	// 将格式化后的字符串输出到某个地方，比如 stdout、file、remote
	_, err := l.out.Write(l.buf)
	return err
}

// Printf calls l.Output to print to the logger.
// Arguments are handled in the manner of fmt.Printf.
// 将参数传递给logger
func (l *Logger) Printf(format string, v ...any) {
	if atomic.LoadInt32(&l.isDiscard) != 0 {
		return
	}
	l.Output(2, fmt.Sprintf(format, v...))
}
~~~

## log 需要提供的方法

我们根据业务使用情况，默认的 logger 可能没法满足我们的需求，比如：

- 设置日志等级
- 设置日志行的格式
- 设置日志的输出目的地
- 最好可以动态改变logger的日志级别，方便找问题

以 Go 的 log 为例
~~~go
// 没有实现等级


// 设置日志行的格式
func (l *Logger) SetFlags(flag int) {
    l.mu.Lock()
    defer l.mu.Unlock()
    l.flag = flag
}

// 设置日志的输出目的地
// SetOutput sets the output destination for the logger.
func (l *Logger) SetOutput(w io.Writer) {
    l.mu.Lock()
    defer l.mu.Unlock()
    l.out = w
    isDiscard := int32(0)
    if w == io.Discard {
        isDiscard = 1
    }
    atomic.StoreInt32(&l.isDiscard, isDiscard)
}
~~~

