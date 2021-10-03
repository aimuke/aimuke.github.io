---
title: "interface 比较"
tags: [go, interface, equal, "相等", "比较"]
---

interface 相等比较的时候常会出现一些比较迷惑的行为，如下：

```go
type T interface {
	f1()
}

func main() {
	test1()
	test2()
}

func test1() {
	var t T
	var i interface{} = t
	fmt.Println(t == nil, i == t, i == nil)
	fmt.Println(reflect.TypeOf(t), reflect.TypeOf(i))
}

func test2() {
	var t *T
	var i interface{} = t
	fmt.Println(t == nil, i == t, i == nil)
	fmt.Println(reflect.TypeOf(t), reflect.TypeOf(i))
}
```

执行上面代码的输出为： 

```bash
true true true
<nil> <nil>
true true false
*main.T *main.T
```

# 原因分析

要明白上述输出的原因，需要了解以下关于interface的概念：

1. interface 包含两个字段 `type` 和`data`，要判断interface相等需要 `type` 和 `data` 都相等。
2. interface 的零值为 `nil`
3. 生成 interface 有两种情况： interface 赋值给 interface， 其他对象赋值给 interface。

> interface 赋值给 interface：
>
> 直接用原变量的 type 字段生成新变量的 type 字段， 原变量的 data 赋值给新变量的 data

> 其他类型赋值给 interface
>
> 使用原变量的类型赋值给 interface 的 type 字段，原变量的值赋值给 interface 的 data 字段。

我们在来分析上面的测试用例：

# test1 分析

先分析 test1 中的过程

 `var t T` 这是生成了一个 interface 变量， go 语言中会直接设置其零值。根据规则1 此时 t 为 nil

`var i interface{} = t` 这一步是将原有的 interface 变量  `t` 赋值给新变量 `i`，直接将 `type` 和 `data`赋值给 `i`，因此赋值后 `i` 的 `type` 和 `data` 与 `t` 的 `type` 和 `data` 都相等。都为 `nil`

# test2 分析

函数 `test2`

`var t *T` 这里定义了一个 `*T` 类型的变量, 这里需要注意的是 变量 `t` 本质上是一个指针，而不是interface。这是 `test2` 与 `test1` 的本质区别。

`var i interface{} = t` 这一步是将指针 `t` 赋值给了一个 `interface i`，这里就涉及到将非 interface 转换为 interface 的过程（`convT2I`），先用 `t` 类型 `*T` 作为 `i` 的 `type`，再用 `t` 的值 `nil` 作为 `i` 的 `data`。因此生成的变量  `i`  的 `type` 是不为 `nil` 的，虽然 `data` 为 `nil`， 根据 `规则1` `i` 不等于  `nil`
