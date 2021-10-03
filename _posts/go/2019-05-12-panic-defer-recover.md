---
title: "defer panic recover"
tags: [go, defer, panic, recover, todo]
---

// TODO 仔细阅读
[Golang: 深入理解panic and recover](https://ieevee.com/tech/2017/11/23/go-panic.html)

# defer
一个 `defer` 调用的函数将被暂时保存到调用列表中. 保存的调用列表在当前环境返回的时候被执行. `defer`一般可以用于简化代码, 执行各种清理操作.

让我们演示一个文件复制的例子: 函数需要打开两个文件, 然后将其中一个文件的内容复制到另一个文件:
```go
func CopyFile(dstName, srcName string) (written int64, err error) {
	src, err := os.Open(srcName)
	if err != nil {
		return
	}

	dst, err := os.Create(dstName)
	if err != nil {
		return
	}

	written, err = io.Copy(dst, src)
	dst.Close()
	src.Close()
	return
}
```
上面的代码虽然能够工作, 但是隐藏一个bug. 如果第二个`os.Open`调用失败, 那么会在没有释放 `source`文件资源的情况下返回. 虽然我们可以通过在第二个返回语句前添加`src.Close()`调用 来修复这个bug; 但是当代码变得复杂时, 类似bug将很难被发现和修复. 通过`defer`语句, 我们可以确保每个文件被关闭:
```go
func CopyFile(dstName, srcName string) (written int64, err error) {
	src, err := os.Open(srcName)
	if err != nil {
		return
	}
	defer src.Close()

	dst, err := os.Create(dstName)
	if err != nil {
		return
	}
	defer dst.Close()

	return io.Copy(dst, src)
}
```
Defer语言可以让我们在打开文件时就思考如何关闭文件. 不管函数如何返回, 文件关闭语句始终会被执行.

Defer语句的行为简单且可预测. 有三个基本原则:
1. 参数在声明时获取
2. 后进先出的顺序执行
3. 可在return语句执行后读取或修改命名的返回值

## 参数值

defer函数的参数值，是在申明defer时确定下来的

在defer函数申明时，对外部变量的引用是有两种方式：
- **函数参数**: 在defer申明时就把值传递给defer，并将值缓存起来，调用defer的时候使用缓存的值进行计算
- **闭包引用**: 在defer函数执行时根据整个上下文确定当前的值

在这个例子中, 表达式`i`的值将在`defer fmt.Println(i)`时被计算. `defer`将会在 当前函数返回的时候打印 `0`.
```go
func a() {
	i := 0
	defer fmt.Println(i)
	i++
	return
}
```

```go
func add(a, b int) int {
    return a + b
}

func test() {
    a, b := 1, 2
    defer fmt.Printf("defer is %d", add(a, b))
    a = 4
    b = 4
    fmt.Printf("test result is %d, %d ", a, b)
}
```
上例中，在`defer`函数被声明时，`defer`函数的参数就会被计算。此处的`defer`函数是 `fmt.Printf`， 其参数为前面的`string`和后面的`add(a,b)`。所以此处实际上是在声明时将`add(a,b)`的结果直接计算出来`3`，然后作为参数。

```go
func t() {
    for i:=0; i<3; i++ {
        defer fmt.Printf("defer param: ", i)
    }

    for i:=0; i<3; i++ {
        defer func() {
            fmt.Printf("closures param: ", i)
        }
    }
}
```
输出为如下，该例中第一个循环，`i`是作为`defer`函数的参数，所以在声明的时候就缓存下来了，输出的是声明时计算出的缓存值。

第二个循环中，`i`是在`defer`函数中作为闭包的引用，在执行时才根据上下文确定当前值。`defer` 执行的时候，循环已经执行完成，所以`i`在defer时是 `3`
```
closures param: 3
closures param: 3
closures param: 3
defer param: 2
defer param: 1
defer param: 0 
```

## 执行顺序
`defer`调用的函数将在当前函数返回的时候, 以后进先出的顺序执行.

下面的函数将输出`3210`:
```go
func b() {
	for i := 0; i < 4; i++ {
		defer fmt.Print(i)
	}
}
```
## 执行时机
`defer`调用的函数可以在返回语句执行后读取或修改命名的返回值.

在这个例子中, `defer`语句将会在当前函数返回后将`i`增加`1`. 实际上, 函数会返回`2`:
```go
func c() (i int) {
	defer func() { i++ }()
	return 1
}
```
`defer` 函数在`return` 语句之后，返回调用函数之前执行。利用该特性, 我们可以方便修改函数的错误返回值. 以后应该可以看到类似的例子.

例1：
```go
func f() (result int) {
    defer func() {
        result++
    }()
    return 0
}
```
例2：
```go
func f() (r int) {
     t := 5
     defer func() {
       t = t + 5
     }()
     return t
}
```

例3：
```go
func f() (r *int) {
     t := 5
     defer func() {
       t = t + 5
     }()
     return *t
}
```
例4：
```go
func f() (r int) {
    defer func(r int) {
          r = r + 5
          fmt.Printf("defer r is %d", r)
    }(r)
    return 1
}
```

**解析：**

例1 return语句中返回值为 0,即将result设置为0， 之后执行defer函数`result++`，result 变为 1. 最终调用函数获取到的返回值为 `1`

例2 返回变量为 `r`，通过`return` 语句将 `r` 设置为 `t`的值`5`. 之后在`defer`函数中修改`t`的值，但是此处修改已经与返回值无关，所以最终的返回值不变还是 `5`

例3 同例2，只是返回值为指针，因此，在`defer`函数中修改`t`，同时也修改了返回值 `r`, 所以返回值是 `10`

例4 首先 `defer` 函数的参数被计算，此处`defer`函数的参数为`r`是`int`的默认值`0`.然后`return`语句设置返回值`r`为 `1`。最后`defer`函数执行，由于`defer`函数的参数是`r`，故在defer函数中执行的`r=r+5`实际上是`defer`函数的局部变量，而不是函数`f（）`的返回值变量。输出为 `defer r is 5`. 同样，这个的修改也不会影响`f()`的返回值.最终返回值为 `1`. 

**下面的代码执行可能有什么问题？**
```go
func f(filenames []string) {
    for _, name := range filenames {
        f, err := os.Open(name)
        if err != nil {
            return err 
        }
        defer f.Close()

        ...
    }
}
```

上面的代码中循环打开文件并进行操作，在循环中设置了`defer`函数来关闭文件。但是由于`defer`函数需要在函数返回时才执行，所以会先打开所有文件并处理，`f()` 返回前才会统一的关闭文件，当文件列表过长时，可能会导致耗光文件描述符。一种解决方式如下:
```go
func f(filenames []string) {
    for _, name := range filenames {
        t(name)
    }
}

func t(name string ) {
    f, err := os.Open(name)
    if err != nil {
        return err 
    }
    defer f.Close()

    ...
}
```
修改为上面之后，在t()执行完之后，文件就会关闭，即一个文件处理完就关闭该文件。这样避免了之前必然存在所有文件都处于打开状态的问题。

# panic
`panic`是一个內建函数，用于停止执行常规控制流程，并开始安全退出。当一个函数`F`调用了`panic`后，`F`将停止执行，任何`F`中的被`defer`开始被正常执行，然后`F`将控制权返回给它的调用者。对于`F`的调用者来说，这时`F`的行为就像是一次`panic`调用一样。这个过程将沿着栈继续往上，直到当前goroutine中的所有函数都返回，此时程序将崩溃。Panics可以通过直接调用`panic`函数发起。它们也可能由运行时错误引起，比如数组访问越界等。

`panic`会停掉**当前**正在执行的程序（注意，不只是协程），但是与`os.Exit(-1)`这种直愣愣的退出不同，`panic`的撤退比较有秩序，他会先处理完当前`goroutine`已经`defer`挂上去的任务，执行完毕后再退出整个程序。

```go
package main

import (
	"os"
	"fmt"
	"time"
)

func main() {
	defer fmt.Println("defer main") // will this be printed when panic?
	var user = os.Getenv("USER_")
	go func() {
        defer fmt.Println("defer caller")
        func() {
            defer func() {
                fmt.Println("defer here")
            }()

            if user == "" {
                panic("should set user env.")
            }
        }()
	}()

    time.Sleep(1 * time.Second)
    fmt.Printf("get result")
}
```

go run一下：
```
defer here
defer caller
```
如上例，`main`中增加一个`defer`，但会睡`1s`，在这个过程中`panic`了，还没等到`main`睡醒，进程已经退出了，因此`main`的`defer`不会被调到；而跟`panic`同一个`goroutine`的`defer caller`还是会打印的，并且其打印在`defer here`之后。

如果把`time.Sleep()`去掉，会发现`main`中的`get result`和`defer main`都调用了，但我觉得这并不是一定的，只是因为并发的存在，`panic`的同时`main`也在往下执行。

**总结：**

- 遇到处理不了的错误，找panic
- panic有操守，退出前会执行本goroutine的defer，方式是原路返回(reverse order)
- panic不多管，不是本goroutine的defer，不执行

# recover
recover也是一个內建函数，它用于重新获取一个`panicking goroutine`的控制权。`recover`只有在被`defer`的函数中才有用。在正常执行期间，调用`recover`将会返回`nil`，并且不会有任何影响。如果当前`goroutine`正在清场，则一次`recover`调用将能够捕获到`panic`的值，并恢复正常执行。


# 需要注意的问题
## defer 定义的位置
`defer` 表达式的函数如果定义在 `panic` 后面，该函数在 `panic` 后就无法被执行到

在defer前panic
```go
func main() {
    panic("a")
    defer func() {
        fmt.Println("b")
    }()
}
```
结果，b没有被打印出来：
```
panic: a

goroutine 1 [running]:
main.main()
    /xxxxx/src/xxx.go:50 +0x39
exit status 2
```
而在defer后panic
```go
func main() {
    defer func() {
        fmt.Println("b")
    }()
    panic("a")
}
```
结果，b被正常打印：
```
b
panic: a

goroutine 1 [running]:
main.main()
    /xxxxx/src/xxx.go:50 +0x39
exit status 2
```
## panic 被捕获
F中出现panic时，F函数会立刻终止，不会执行F函数内panic后面的内容，但不会立刻return，而是调用F的defer，如果F的defer中有recover捕获，则F在执行完defer后正常返回，调用函数F的函数G继续正常执行

```go
func G() {
    defer func() {
        fmt.Println("c")
    }()
    F()
    fmt.Println("继续执行")
}

func F() {
    defer func() {
        if err := recover(); err != nil {
            fmt.Println("捕获异常:", err)
        }
        fmt.Println("b")
    }()
    panic("a")
}
```
结果：
```
捕获异常: a
b
继续执行
c
```

## panic 未捕获
如果F的defer中无recover捕获，则将panic抛到G中，G函数会立刻终止，不会执行G函数内后面的内容，但不会立刻return，而调用G的defer...以此类推
```go
func G() {
    defer func() {
        if err := recover(); err != nil {
            fmt.Println("捕获异常:", err)
        }
        fmt.Println("c")
    }()
    F()
    fmt.Println("继续执行")
}

func F() {
    defer func() {
        fmt.Println("b")
    }()
    panic("a")
}
```
结果：
```
b
捕获异常: a
c
```

## panic 的范围
如果一直没有recover，抛出的panic到当前goroutine最上层函数时，程序直接异常终止
```go
func G() {
    defer func() {
        fmt.Println("c")
    }()
    F()
    fmt.Println("继续执行")
}

func F() {
    defer func() {
        fmt.Println("b")
    }()
    panic("a")
}
```
结果：
```
b
c
panic: a

goroutine 1 [running]:
main.F()
    /xxxxx/src/xxx.go:61 +0x55
main.G()
    /xxxxx/src/xxx.go:53 +0x42
exit status 2
```

## recover 的范围
recover都是在当前的goroutine里进行捕获的，这就是说，对于创建goroutine的外层函数，如果goroutine内部发生panic并且内部没有用recover，外层函数是无法用recover来捕获的，这样会造成程序崩溃
```go
func G() {
    defer func() {
        //goroutine外进行recover
        if err := recover(); err != nil {
            fmt.Println("捕获异常:", err)
        }
        fmt.Println("c")
    }()
    //创建goroutine调用F函数
    go F()
    time.Sleep(time.Second)
}

func F() {
    defer func() {
        fmt.Println("b")
    }()
    //goroutine内部抛出panic
    panic("a")
}
```
结果：
```
b
panic: a

goroutine 5 [running]:
main.F()
    /xxxxx/src/xxx.go:67 +0x55
created by main.main
    /xxxxx/src/xxx.go:58 +0x51
exit status 2
```

## recover 返回值

recover返回的是interface{}类型而不是go中的 error 类型，如果外层函数需要调用err.Error()，会编译错误，也可能会在执行时panic
```go
func main() {
    defer func() {
        if err := recover(); err != nil {
            fmt.Println("捕获异常:", err.Error())
        }
    }()
    panic("a")
}
```
编译错误，结果：
```
err.Error undefined (type interface {} is interface with no methods)
```

```go
func main() {
    defer func() {
        if err := recover(); err != nil {
            fmt.Println("捕获异常:", fmt.Errorf("%v", err).Error())
        }
    }()
    panic("a")
}
```
结果：
```
捕获异常: a
```

```go
package main

import(
	"fmt"
	"reflect"
)

func main()  {
    defer func() {
        if err:=recover();err!=nil{
            f:=err.(func()string)
            fmt.Println(err,f(),reflect.TypeOf(err).Kind().String())
        }
    }()
    panic(func() string {
            return  "main panic"
        })
}
```

执行
```
0x47d9c0 main panic func
```

## 多个panic

```go
package main
func main() {
	defer func() {
		panic("defer panic")
	}()
	panic("main panic")
}
```

运行
```
panic: main panic
        panic: defer panic
...省略
```

```go
package main

import(
	"fmt"
)

func main()  {
	test()
	fmt.Println("main end")
}

func test() {
	defer func() {
        if err:=recover();err!=nil{
            fmt.Println(err)
        }
    }()

    defer func() {
        panic("defer panic")
    }()
    panic("test panic")
}
```

执行
```
defer panic
main end
```
当recover前有多个panic时， 捕获到的只有最后一个.且捕获后，程序不会退出，而是正常运行。

# 参考文献

- [Defer, Panic, and Recover[翻译]](https://my.oschina.net/chai2010/blog/119216)

- [Defer, Panic, and Recover](https://blog.golang.org/defer-panic-and-recover)

- [go defer,panic,recover详解 go 的异常处理](https://www.jianshu.com/p/63e3d57f285f)

- [Golang: 深入理解panic and recover](https://ieevee.com/tech/2017/11/23/go-panic.html)
