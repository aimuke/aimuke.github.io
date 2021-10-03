---
title: "[译]像牛人一样改进你的Go代码"
tags: [go]
---

原文: [Lint your #golang code like a mad man!](https://medium.com/@arshamshirvani/lint-your-golang-code-like-a-pro-668dc6637b39), 作者: [Arsham Shirvani](https://medium.com/@arshamshirvani)

我使用下面的工具来改进我的代码，除了 `vendor` 文件夹。我的操作系统是GNU/Linux,但是稍微修改一下脚本应该也能运行在你的操作系统上。我使用 `glide` 来处理依赖(`vendor`),但你也可以使用你的包依赖管理工具来替换 `glide nv`， 这个命令列出了所有的文件夹，除了 `vender` (译者按： Go 1.9中可以直接使用 `./...` ，它会排除 `vendor` 文件夹)。有些情况下 `glide nv` 不适合，所以我使用了它的老式风格。

注意我使用 `$` 作为shell的提示符。

# gofmt
Go安装程序中自带了 `gofmt` 工具，可以使用它来格式化代码，保持一致的代码风格：

```sh
$ find . -name "*.go" -not -path "./vendor/*" -not -path ".git/*" | xargs gofmt -s -d
```

# gocyclo

gocyclo 用来检查函数的复杂度。

安装：
```sh
$ go get -u github.com/fzipp/gocyclo
```
使用：
```sh
$ gocyclo -over 12 $(ls -d */ | grep -v vendor)
```

上面的命令列出了所有复杂度大于12的函数。你还可以提出最复杂的几个：

```sh
$ gocyclo -top 10 $(ls -d */ | grep -v vendor)
```

# deadcode

`deadcode` 会告诉你哪些代码片段根本没用。

安装:

```sh
$ go get -u github.com/tsenart/deadcode
```

使用：

```sh
$ find . -type d -not -path "./vendor/*" | xargs deadcode
```

# gotype

`gotype` 会对go文件和包进行语义(`semantic`)和句法(`syntactic`)的分析,这是google提供的一个工具。

安装：

```sh
$ go get -u golang.org/x/tools/cmd/gotype
```

使用：

```sh
$ find . -name "*.go" -not -path "./vendor/*" -not -path ".git/*" -print0 | xargs -0 gotype -a
```

# misspell

`misspell` 用来拼写检查，对国内英语不太熟练的同学很有帮助。

安装:

```sh
$ go get -u github.com/client9/misspell/cmd/misspell
```

使用:

```sh
$ find . -type f -not -path "./vendor/*" -print0 | xargs -0 misspell
```

# staticcheck

`staticcheck` 是一个超牛的工具，提供了巨多的静态检查，就像 C#生态圈的 `ReSharper` 一样。

安装：

```sh
$ go get -u honnef.co/go/staticcheck/cmd/staticcheck
```

使用：

```sh
$ staticcheck $(glide nv)
```

> 译者按：`staticcheck` 工具已经合并到了项目：[go-tools](https://github.com/dominikh/go-tools) ，这个项目提供了非常好的工具， 还包括 `structlayout-optimize`、`unused`、`rdeps`、`keyify`等，值的你去探索。

# goconst

`goconst` 会查找重复的字符串，这些字符串可以抽取成常量。

```sh
$ go get -u github.com/jgautheron/goconst/cmd/goconst
```

使用:

```sh
$ goconst ./… | grep -v vendor
```

以上是作者列出的一些工具， 和我以前的一篇文章中列出的工具有很多重合的： 使用工具检查你的代码， 事实上我在项目中已经使用了文中很多的代码，非常非常的有帮助，希望你在阅读后能有所收获，快将这些工具加入到你的Makefile文件中吧。

# References

- [原文 Lint your #golang code like a mad man!](https://medium.com/@arshamshirvani/lint-your-golang-code-like-a-pro-668dc6637b39)

- [[译]像牛人一样改进你的Go代码](https://colobu.com/2017/06/27/Lint-your-golang-code-like-a-mad-man/)
