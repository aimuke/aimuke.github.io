---
title: "go语言静态库的编译和使用"
tags: [go, build, static]
---

[原文 - go语言静态库的编译和使用](http://reborncodinglife.com/2018/04/27/how-to-create-static-lib-in-golang/)

本文主要介绍go语言静态库的编译和使用方法，以windows平台为例，linux平台步骤一样，具体环境如下：

```sh
>echo %GOPATH%
E:\share\git\go_practice\

>echo %GOROOT%
C:\Go\

>tree /F %GOPATH%\src
卷 work 的文件夹 PATH 列表
卷序列号为 0009-D8C8
E:\SHARE\GIT\GO_PRACTICE\SRC
│  main.go
│
└─demo
        demo.go

```

在 `%GOPATH%\src` 目录，有demo包和使用demo包的应用程序 `main.go` ， `main.go` 代码如下：

```go
package main

import "demo"

func main() {
    demo.Demo()
}
```

demo 包中的 `demo.go` 代码如下：

```go
package demo

import "fmt"

func Demo() {
    fmt.Println("call demo ...")
}
```

由于 `demo.go` 是 `%GOPATH%\src` 目录下的一个包， `main.go` 在import该包后，可以直接使用，运行 `main.go`：

```sh
>go run main.go
call demo ...
```

现在，需要将 `demo.go` 编译成静态库 `demo.a`，不提供`demo.go`的源代码，让`main.go`也能正常编译运行，详细步骤如下：

# 1 编译静态库demo.a

```sh
>go install demo
```

在命令行运行`go install demo`命令，会在`%GOPATH%`目录下生相应的静态库文件`demo.a`（windows平台一般在`%GOPATH%\src\pkg\windows_amd64`目录）。

# 2 编译main.go

进入 `main.go` 所在目录，编译 `main.go`：

```sh
>go tool compile -I E:\share\git\go_practice\pkg\windows_amd64 main.go
```

`-I` 选项指定了 `demo` 包的安装路径，供 `main.go` 导入使用，即 `E:\share\git\go_practice\pkg\win dows_amd64` 目录，编译成功后会生成相应的目标文件 `main.o`。

# 3 链接main.o

```sh
>go tool link -o main.exe -L E:\share\git\go_practice\pkg\windows_amd64 main.o
```

`-L` 选项指定了静态库 `demo.a` 的路径，即 `E:\share\git\go_practice\pkg\win dows_amd64` 目录，链接成功后会生成相应的可执行文件 `main.exe`。

# 4 运行main.exe

```sh
>main.exe
call demo ...
```

现在，就算把 `demo` 目录删除，再次编译链接 `main.go`，也能正确生成 `main.exe`:

```sh
>go tool compile -I E:\share\git\go_practice\pkg\windows_amd64 main.go

>go tool link -o main.exe -L E:\share\git\go_practice\pkg\windows_amd64 main.o

>main.exe
call demo ...
```

但是，如果删除了静态库 `demo.a` ，就不能编译 `main.go`，如下：

```sh
>go tool compile -I E:\share\git\go_practice\pkg\windows_amd64 main.go
main.go:3: can't find import: "demo"
```

以上就是go语言静态库的编译和使用方法，下次介绍 [动态库的编译和使用方法]((http://reborncodinglife.com/2018/04/29/how-to-create-dynamic-lib-in-golang/))。

# References

- [原文 - go语言静态库的编译和使用](http://reborncodinglife.com/2018/04/27/how-to-create-static-lib-in-golang/)

- [go语言动态库的编译和使用](http://reborncodinglife.com/2018/04/29/how-to-create-dynamic-lib-in-golang/)
