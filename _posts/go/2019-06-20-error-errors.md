---
title: "Go如何优雅地错误处理(Error Handling and Go 1)"
tags: [go, error, errors]
---

Go的错误处理一直被吐槽太繁琐, 作为主要用GO的攻城狮, 经常写 `if err!=nil`, 但是如果想偷懒, 少带了上下文信息, 直接写 `if err!=nil { return err}` 或者 `fmt.Errorf` 携带的上下文信息太少了的话, 看到错误日志也会一脸懵逼, 难以定位问题.

官方在 2011 年就发过一篇博客教大家如何[在Go中处理error](https://blog.golang.org/error-handling-and-go) , error 是一个内建的 interface, 鼓励大家用好自定义错误类型, 常用的范式有三种:

- 一是用 `errors.New(str string)` 定义错误常量, 让调用方去判断返回的 err 是否等于这个常量, 来进行区分处理;
- 二是用 `fmt.Errorf(fmt string, args... interface{})` 增加一些上下文信息, 用文字的方式告诉调用方哪里出错了, 让调用方打错误日志出来;
- 三是自定义 `struct type` , 实现 error 接口, 调用方用类型断言转成特定的 `struct type` , 拿到更结构化的错误信息.

我最开始最常用的做法是, `fmt.Errorf` 时写上此函数函数名、调用出错的函数名、参数是什么、err , 代码十分啰嗦, 而且通常打日志是在上层函数打的, 看到错误日志还需要用函数名去代码中搜索看看在哪里出错. 业务代码调用层级一多,非常麻烦. 很多情况下我既想带上下文信息, 又想在上层调用方取得最里层出错的函数返回的error常量或自定义的 struct type, 最好还能自动带上行号函数名信息, 减少每次写 `fmt.Errof` 的手动写上函数名的痛苦. 于是开始在 github 找包, star 数最高的是 `pkg/errors` 、`juju/errors`.

# pkg/errors

`pkg/errors` 解决了一些问题, 核心函数是 `Wrapf` 和 `Cause`: 
- `Wrapf` 包装错误附加上下文信息并带上调用栈, 但是每次去包装错误的时候都去取一次调用栈, 完全没有必要啊, 因为最早出错的函数里就能拿到完整的调用栈的, 并且调用栈打出来的信息也不好看, 而且通常HTTP服务会用框架, 用了框架的话调用栈就会肿起来, 这些框架的固定调用栈信息打印出来毫无帮助. 
- `Cause` 去递归拿到最里层的 error, 用于和error常量比较或类型断言成自定义 struct type.

**Wrapf**
```go
// Wrapf returns an error annotating err with a stack trace
// at the point Wrapf is call, and the format specifier.
// If err is nil, Wrapf returns nil.
func Wrapf(err error, format string, args ...interface{}) error {
	if err == nil {
		return nil
	}
	err = &withMessage{
		cause: err,
		msg:   fmt.Sprintf(format, args...),
	}
	return &withStack{
		err,
		callers(),
	}
}
```
**Cause**
```go
// Cause returns the underlying cause of the error, if possible.
// An error value has a cause if it implements the following
// interface:
//
//     type causer interface {
//            Cause() error
//     }
//
// If the error does not implement Cause, the original error will
// be returned. If the error is nil, nil will be returned without further
// investigation.
func Cause(err error) error {
	type causer interface {
		Cause() error
	}
	for err != nil {
		cause, ok := err.(causer)
		if !ok {
			break
		}
		err = cause.Cause()
	}
	return err
}
```

# juju/errors
juju/errors API非常复杂, 包装的error的函数就有三个 
- func Annotatef(other error, format string, args ...interface{}) error 
- func Maskf(other error, format string, args ...interface{}) error 
- func Wrapf(other, newDescriptive error, format string, args ...interface{}) error

每次包装时都会 `SetLocation` , 消耗更大, 即使有时不需要打印`error string` 只需要判断, 它也去用`runtime.Caller`去拿文件名, 行号; 调用栈打出来的信息也不好看.

```go
// SetLocation records the source location of the error at callDepth stack
// frames above the call.
func (e *Err) SetLocation(callDepth int) {
	_, file, line, _ := runtime.Caller(callDepth + 1)
	e.file = trimGoPath(file)
	e.line = line
}
```

# 重写error
以上包不满足要求, 只能造轮子了. 两个思想. API要设计的简单, 调用栈要好看 https://github.com/hanjm/errors

- API简单: 定义error常量只有 errors.New 函数, 兼容标准库的函数, 兼容很重要；包装error的只有 errors.Errorf 函数, 只在最早出错的时候取调用栈, 调用方再包装时无需取调用栈, 此时只需要pc, 不需要这时就把文件名行号取出来; 取最里层的 error 只有 errors.GetInnerMost, 用于和 error 常量比较或类型断言成自定义 struct type分类处理.
- 调用栈好看: 去掉标准包的调用栈, 去掉框架固定的调用栈信息(通常是github.com的包), 只保留业务逻辑的调用栈. 按`[ 文件名:行号 函数名:message]`分行格式化输出, 把调用栈和附加的message对应起来. (第一版格式是`[文件名:行号 函数名:message]`, 没有空格, 后面有个同事说**在Goland IDE里看panic信息时可以点击定位到源码**, 你的包能不能加这个功能, 所以去研究了下, 写了几个print的demo试了下发现如果输出中的文件名前后带空格的话, intellij IDE会自动识别输出中的文件名变成超链接, 所以给 “文件名:行号” 前后加了空格, 就能在IDE中直接点击定位到源码对应的行, 非常地方便, 感谢这位同事)

在IDE中加个live template, 写errf回车就补全到

{% raw %}
```go
if err!=nil {
	err = errors.Errorf(err,"{{光标}}")
	return
}
```
{% endraw %}

然后补充必要的注释和参数就行了, 在本地环境调试时看到错误日志点击就可以定位到源码, 在非本地环境跑看到错误日志相比之前也能更好地知道发生了什么, 复制文件名:行号到IDE中就能定位到源码, 大大减轻了错误处理的繁琐.

# 参考文献

- [原文地址](https://imhanjm.com/2018/07/08/go%E5%A6%82%E4%BD%95%E4%BC%98%E9%9B%85%E5%9C%B0%E9%94%99%E8%AF%AF%E5%A4%84%E7%90%86(error%20handling%20and%20go%201)/)
