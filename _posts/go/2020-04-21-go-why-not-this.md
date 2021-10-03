---
title: "为什么不建议使用 this作为方法接受符"
tags: [go, receiver, this, self]
---

在用 `go` 编程为 `struct` 添加方法的时候，我们常常会习惯性的将方法的接收符写为 `this`。这样很符合以前的编程习惯，而且 `this` 在方法中一看就很明显的代表了当前对象。

```go
type P struct {
	// ...
}

func (this P) Fun() {
	// ...
}
```

这时候一般 IDE 会提示一些警告信息，我用的vscode 里面提示的是:

> receiver name should be a reflection of its identity; don't use generic names such as "this" or "self"
>
> **接收符应该反映它自己的身份，不要使用 `this` 或 `self` 这种通用的名字**

这种不建议使用通用标示的情况和我们以前的编程习惯有些不同，于是我查找了一下是否有相关的原因。

# 官网说明

在官网的wiki中 [go CodeReviewComments](https://github.com/golang/go/wiki/CodeReviewComments#receiver-names) 中有一条关于 `receiver-name` 的说明：

> The name of a method's receiver should be a reflection of its identity; often a one or two letter abbreviation of its type suffices (such as "c" or "cl" for "Client"). Don't use generic names such as "me", "this" or "self", identifiers typical of object-oriented languages that gives the method a special meaning. In Go, the receiver of a method is just another parameter and therefore, should be named accordingly. The name need not be as descriptive as that of a method argument, as its role is obvious and serves no documentary purpose. It can be very short as it will appear on almost every line of every method of the type; familiarity admits brevity. Be consistent, too: if you call the receiver "c" in one method, don't call it "cl" in another.

这段话说了几个意思：

1. 名称反应了其角色，通常用缩写的一两个字母就可以了；
2. 方法的接收符不应该使用使用一些通用的名称（`generic names`）,因为这些典型的面向对象的标示符常会给与方法一些特殊的含义。
3. go 中接收符只是一个普通的参数，因此应该像其他一些参数那样进行命名
4. 由于其角色很明显，而且没有文档化作用，所以可以用一个很短的缩写就就可以了。
5. 为了一致性，最好每个方法的接收符都用相同的名称

# 自己理解

由于长时间以来，都习惯于将方法的接收符叫做 `this`。实在很不习惯 go 里面这个不用 `this` 的方式。在网上查了很多资料，好像也没有找到具有说服性的原因。现将看到的一些说法做一个记录，待以后参考。

## this 的含义

go 与以前接触的语言最大的一个不同在于，go 是值传递。其他语言主要是指针传递。在 oop 的其他语言中， `this` 往往指代的是对象自身指针，对 `this` 的修改就相当于对 `this` 指针对应的对象进行了修改。

但是在go中由于是使用值传递的方式，面对两种不同的接收符（对象和对象指针）。如果都使用 `this` 的话，其实是不对的。因为使用对象作为接收符的时候，这里的 `this` 其实代表不了其指针对应的对象，对其修改也不能反映到原本的对象上。如下：

```go
type P struct {
	x int
}

func (p P) add(i int) {
	p.x = p.x + i
}

func main() {
	p := P{0}
	fmt.Printf("before %d \n", p.x)
	p.add(1)
	fmt.Printf("after %d \n", p.x)
}
```

输出为： 
```
before 0
after 0
```

可以看到此时如果将接收符命名为 `this` 的话，我们其实是不对的，这里由于是值传递，修改不会在原对象上呈现。违反了以前 OOP 中的直觉，很容易出错。

因此，**由于 `this` 默认代表了指针的含义，在使用对象作为接收符的方法中，应该是不适宜用 `this` 作为接收符的。**

## go 的方法

go 里面的方法本质上都是定义在 `type` 上的，接收符只是指明了其方法的一个参数而已。比如：

```go
type P struct {
	x int
}

func (p P) Get() int {
	return p.x
}

func (p *P) Set(i int) {
	p.x = i
}

func main() {
	p := P{0}
	ret := P.Get(p) // 等同于 p.Get()
	fmt.Printf("after %d \n", ret)
	(*P).Set(&p, 1) // 等同于 p.Set(1)
	ret = P.Get(p)
	fmt.Printf("after %d \n", ret)
}
```

当我们使用 type 来对方法进行调用的时候，更加能感觉到接收符就是普通参数的含义。说明中的第3条：

> 3. go 中接收符只是一个普通的参数，因此应该像其他一些参数那样进行命名

虽然从参数上来说，接收符和普通参数没有区别， 因此建议使用普通的参数命名。

但是我感觉在指针作为接收符的时候，其实使用 `this` 是一个更合适的选择。方便明确，一下就能看出其代表的是自身对象。

# 其他解释

[ANDREY PETROV](https://blog.heroku.com/authors/andrey-petrov) 的文章 [Neither self nor this: Receivers in Go](https://blog.heroku.com/neither-self-nor-this-receivers-in-go) 在关于 receiver 命名的讨论中被多次提到。

他再文章中说到，go 里面的方法是可以独立存在而不依赖与接收符的，比如:

```go
withReceiver := myServer.Name
without := Server.Name
```

上面定义的两个变量，一个使用了接收符，一个没有使用接收符。这两个变量都可以任意的传递。当没有接收符的变量在执行的时候需要传入一个对象实例作为参数，这个对象实例这时其实就是一个普通的参数。因此没有必要使用 `this`.

另外他还提了一个很有意思的观点，是说如果你都用一些通用的名称作为接收符，那么在重构的时候，可能会由于接收符相同而引起错误。这个观点我是持保留态度的，原因如下：

- 如果这个能成为一个问题的话，那么其他的 OOP 语言都应该放弃使用 `this`
- 重构有很多工具可以避免这个问题
- 假设工具无法避免，那么就算不用 `this` 也只是减小了概率而已，在一个大的项目中，重名的机会依然很大。

# References

- [go CodeReviewComments](https://github.com/golang/go/wiki/CodeReviewComments#receiver-names)

- [Neither self nor this: Receivers in Go](https://blog.heroku.com/neither-self-nor-this-receivers-in-go)
