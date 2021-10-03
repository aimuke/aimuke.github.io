---
title: "Go 语言切片的三种特殊状态"
tags: [go, slice]
---

> **摘要**： 我们今天要来讲一个非常细节的小知识，这个知识被大多数 Go 语言的开发者无视了，它就是切片的三种特殊状态: 「`零切片`」、「`空切片`」和「`nil 切片`」。 图片切片被视为 Go 语言中最为重要的基础数据结构，使用起来非常简单，有趣的内部结构让它成了 Go 语言面试中最为常见的考点。

![slice-struct](/assets/images/2019/0530/slice-status-struct.jpg)

切片被视为 Go 语言中最为重要的基础数据结构，使用起来非常简单，有趣的内部结构让它成了 Go 语言面试中最为常见的考点。切片的底层是一个数组，切片的表层是一个包含三个变量的结构体，当我们将一个切片赋值给另一个切片时，本质上是对切片表层结构体的浅拷贝。结构体中第一个变量是一个指针，指向底层的数组，另外两个变量分别是切片的长度和容量。

```go
type slice struct {
	array unsafe.Pointer
	length int
	capcity int
}
```
# zero slice
我们今天要讲的特殊状态之一「`零切片(zero slice)`」其实并不是什么特殊的切片，它只是表示底层数组的二进制内容都是零。比如下面代码中的 `s` 变量就是一个「`零切片`」
```go
var s = make([]int, 10)

fmt.Println(s)
```

```
[0 0 0 0 0 0 0 0 0 0]
```
如果是一个指针类型的切片，那么底层数组的内容就全是 `nil`
```go
var s = make([]*int, 10)

fmt.Println(s)
```

```
[<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil>]
```
零切片还是比较易于理解的，这部分我也就不再以钻牛角尖的形式继续自我拷问。

# empty and nil slice
下面我们要引入「`空切片`」和 「`nil 切片`」，在理解它们的区别之前我们先看看一个长度为零的切片都有那些形式可以创建出来
```go
var s1 []int
var s2 = []int{}
var s3 = make([]int, 0)
// new 函数返回是指针类型，所以需要使用 * 号来解引用
var s4 = *new([]int)

fmt.Println(len(s1), len(s2), len(s3), len(s4))
fmt.Println(cap(s1), cap(s2), cap(s3), cap(s4))
fmt.Println(s1, s2, s3, s4)
```

```
0 0 0 0
0 0 0 0
[] [] [] []
```

## 指针值不同
上面这四种形式从输出结果上来看，似乎一摸一样，没区别。但是实际上是有区别的，我们要讲的两种特殊类型「`空切片`」和「 `nil 切片`」，就隐藏在上面的四种形式之中。

我们如何来分析三面四种形式的内部结构的区别呢？接下里要使用到 Go 语言的高级内容，通过 `unsafe.Pointer` 来转换 Go 语言的任意变量类型。

因为切片的内部结构是一个结构体，包含三个机器字大小的整型变量，其中第一个变量是一个指针变量，指针变量里面存储的也是一个整型值，只不过这个值是另一个变量的内存地址。我们可以将这个结构体看成长度为 `3` 的整型数组 `[3]int`。然后将切片变量转换成 `[3]int`。
```go
var s1 []int
var s2 = []int{}
var s3 = make([]int, 0)
var s4 = *new([]int)

var a1 = *(*[3]int)(unsafe.Pointer(&s1))
var a2 = *(*[3]int)(unsafe.Pointer(&s2))
var a3 = *(*[3]int)(unsafe.Pointer(&s3))
var a4 = *(*[3]int)(unsafe.Pointer(&s4))

fmt.Println(a1)
fmt.Println(a2)
fmt.Println(a3)
fmt.Println(a4)
```

```
[0 0 0]
[824634199592 0 0]
[824634199592 0 0]
[0 0 0]
```

从输出中我们看到了明显的神奇的让人感到意外的难以理解的不一样的结果。如果上面的 `unsafe` 代码你不能理解，那就继续等等我的《快学 Go 语言》章节的更新吧。

其中输出为 `[0 0 0]` 的 `s1` 和 `s4` 变量就是「 `nil 切片`」，`s2` 和 `s3` 变量就是「`空切片`」。`824634199592` 这个值是一个特殊的内存地址，所有类型的「`空切片`」都共享这一个内存地址。下面的代码中三个空切片都指向了同一个内存地址。
```go
var s2 = []int{}
var s3 = make([]int, 0)

var a2 = *(*[3]int)(unsafe.Pointer(&s2))
var a3 = *(*[3]int)(unsafe.Pointer(&s3))

fmt.Println(a2)
fmt.Println(a3)

var s5 = make([]struct{ x, y, z int }, 0)
var a5 = *(*[3]int)(unsafe.Pointer(&s5))

fmt.Println(a5)
```

```
[824634158720 0 0]
[824634158720 0 0]
[824634158720 0 0]
```
用图形来表示「`空切片`」和「 `nil 切片`」如下

![nil and empty slice](/assets/images/2019/0530/slice-status-nil-empty.jpg)

空切片指向的 `zerobase` 内存地址是一个神奇的地址，从 Go 语言的源代码中可以看到它的定义
```go
//// runtime/malloc.go

// base address for all 0-byte allocations
var zerobase uintptr

// 分配对象内存
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
   ...
   if size == 0 {
      return unsafe.Pointer(&zerobase)
   }
   ...
}

//// runtime/slice.go
// 创建切片
func makeslice(et *_type, len, cap int) slice {
   ...
   p := mallocgc(et.size*uintptr(cap), et, true)
   return slice{p, len, cap}
}
```
## 使用区别
最后一个问题是：「 `nil 切片`」和 「`空切片`」在使用上有什么区别么？

答案是完全没有任何区别！No！不对，还有一个小小的区别！请看下面的代码
```go
package main

import "fmt"

func main() {
   var s1 []int
   var s2 = []int{}

   fmt.Println(s1 == nil)
   fmt.Println(s2 == nil)
   fmt.Printf("%#v\n", s1)
   fmt.Printf("%#v\n", s2)
}
```

```
true
false
[]int(nil)
[]int{}
```

所以为了避免写代码的时候把脑袋搞昏的最好办法是不要创建「 `空切片`」，统一使用「 `nil 切片`」，同时要避免将切片和 `nil` 进行比较来执行某些逻辑。这是官方的标准建议。

> The former declares a nil slice value, while the latter is non-nil but zero-length. They are functionally equivalent—their len and cap are both zero—but the nil slice is the preferred style.

## 与 nil 比较
「`空切片`」和「 `nil 切片`」有时候会隐藏在结构体中，这时候它们的区别就被太多的人忽略了，下面我们看个例子
```go
type Something struct {
   values []int
}

var s1 = Something{}
var s2 = Something{[]int{}}

fmt.Println(s1.values == nil)
fmt.Println(s2.values == nil)
```

```
true
false
```
可以发现这两种创建结构体的结果是不一样的！第一种无参构造创建了 `nil 切片`，而第二种则创建了`空切片`。

## json 序列化结果
「`空切片`」和「 `nil 切片`」还有一个极为不同的地方在于 `JSON` 序列化

```go
type Something struct {
	Values []int
}

var s1 = Something{}
var s2 = Something{[]int{}}

bs1, _ := json.Marshal(s1)
bs2, _ := json.Marshal(s2)

fmt.Println(string(bs1))
fmt.Println(string(bs2))
```

```
{"Values":null}

{"Values":[]}
```
Ban! Ban! Ban! 它们的 `json` 序列化结果居然也不一样！


# 参考文献

- [原文地址](https://zhuanlan.zhihu.com/p/49529590)


