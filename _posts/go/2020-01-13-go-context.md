---
title: "Golang Context实现与使用"
tags: [go, context, gorouting, chan, waitgroup]
---

# 背景
go 语言中对并发的使用很多，常会出现一个 gorouting 中发起多个子 gorouting。然后子 gorouting在发起子gorouting的情况。当系统中存在多个有层级的gorouting时，就需要对gorouting进行统一的控制，让gorouting在适当的时候自动退出或执行相应的操作。

## 协程结构与关系

程序中发起新的协程的操作是一个典型的树状结构。即有父协程发起子协程，子协程在发起自己的子协程。每一个协程与自己发起的协程及其后代都构成树状的结构。

对这些树状的协程进行控制，主要有一下几种场景：

- `主协程主动结束` 主协程结束后，所有后代都结束。比如：主协程开启多个子协程分别执行任务，主协程接到外部停止命令后，所有子协程都需要停止。
- `主协程被动等待` 由后代子协程决定什么时候需要停止，又分为两种情况：
   - 需要所有的子节点都结束后，主协程才能结束。比如：分布式结束，需要每一个子任务都计算完成了，主任务才能结束
   - 某子节点结束后整个结束。比如：某个事物由多个子协程操作组成，任意一个失败，则整个任务都失败。
- 还有其他的情况，在这里不做讨论

## WaitGroup

`WaitGroup` 以前我们在并发的时候介绍过，它是一种控制并发的方式，它的这种方式是控制多个 `goroutine` 同时完成。即上面提到的主协程被动等待所有子协程完成这种场景:

```go
func main() {
	var wg sync.WaitGroup

	wg.Add(2)
	go func() {
		time.Sleep(2*time.Second)
		fmt.Println("1号完成")
		wg.Done()
	}()
	go func() {
		time.Sleep(2*time.Second)
		fmt.Println("2号完成")
		wg.Done()
	}()
	wg.Wait()
	fmt.Println("好了，大家都干完了，放工")
}
```

一个很简单的例子，一定要例子中的2个goroutine同时做完，才算是完成，先做好的就要等着其他未完成的，所有的goroutine要都全部完成才可以。

这是一种控制并发的方式，这种尤其适用于，好多个goroutine协同做一件事情的时候，因为每个goroutine做的都是这件事情的一部分，只有全部的goroutine都完成，这件事情才算是完成，这是等待的方式。

在实际的业务种，我们可能会有这么一种场景：需要我们主动的通知某一个goroutine结束。比如我们开启一个后台goroutine一直做事情，比如监控，现在不需要了，就需要通知这个监控goroutine结束，不然它会一直跑，就泄漏了。

## chan

我们都知道一个goroutine启动后，我们是无法控制他的，大部分情况是等待它自己结束，那么如果这个goroutine是一个不会自己结束的后台goroutine呢？比如监控等，会一直运行的。

这种情况化，一直傻瓜式的办法是全局变量，其他地方通过修改这个变量完成结束通知，然后后台goroutine不停的检查这个变量，如果发现被通知关闭了，就自我结束。

这种方式也可以，但是首先我们要保证这个变量在多线程下的安全，基于此，有一种更好的方式：`chan + select` 。

```go
func main() {
	stop := make(chan struct{})

	go func() {
		for {
			select {
			case <-stop:
				fmt.Println("监控退出，停止了...")
				return
			default:
				fmt.Println("goroutine监控中...")
				time.Sleep(2 * time.Second)
			}
		}
	}()

	time.Sleep(10 * time.Second)
	fmt.Println("可以了，通知监控停止")
	close(stop)
	//为了检测监控过是否停止，如果没有监控输出，就表示停止了
	time.Sleep(5 * time.Second)

}
```

例子中我们定义一个 `stop` 的 `chan` ，通知他结束后台goroutine。实现也非常简单，在后台goroutine中，使用 `select` 判断 `stop` 是否可以接收到值，如果可以接收到，就表示可以退出停止了；如果没有接收到，就会执行 `default` 里的监控逻辑，继续监控，只到收到 `stop` 的通知。

有了以上的逻辑，我们就可以在其他goroutine种，给 `stop chan` 发送值或者关闭 `stop`，例子中是在 `main` goroutine中发送的，控制让这个监控的goroutine结束。

发送了 `close(stop)` 结束的指令后，我这里使用 `time.Sleep(5 * time.Second)` 故意停顿5秒来检测我们结束监控goroutine是否成功。如果成功的话，不会再有 `goroutine监控中...` 的输出了；如果没有成功，监控goroutine就会继续打印 `goroutine监控中...` 输出。

这种 `chan+select` 的方式，是比较优雅的结束一个goroutine的方式。

## Context

比如一个网络请求Request，每个Request都需要开启一个goroutine做一些事情，这些goroutine又可能会开启其他的goroutine。所以我们需要一种可以跟踪goroutine的方案，才可以达到控制他们的目的，这就是Go语言为我们提供的 `Context` ，称之为上下文非常贴切，它就是goroutine的上下文。

下面我们就使用Go Context重写上面的示例。

```go
func main() {
	ctx, cancel := context.WithCancel(context.Background())
	go func(ctx context.Context) {
		for {
			select {
			case <-ctx.Done():
				fmt.Println("监控退出，停止了...")
				return
			default:
				fmt.Println("goroutine监控中...")
				time.Sleep(2 * time.Second)
			}
		}
	}(ctx)

	time.Sleep(10 * time.Second)
	fmt.Println("可以了，通知监控停止")
	cancel()
	//为了检测监控过是否停止，如果没有监控输出，就表示停止了
	time.Sleep(5 * time.Second)

}
```

重写比较简单，就是把原来的 `chan stop` 换成 `Context` ，使用 `Context` 跟踪goroutine，以便进行控制，比如结束等。

`context.Background()` 返回一个空的 `Context` ，这个空的 `Context` 一般用于整个 `Context` 树的根节点。然后我们使用 `context.WithCancel(parent)` 函数，创建一个可取消的子 `Context` ，然后当作参数传给goroutine使用，这样就可以使用这个子 `Context` 跟踪这个goroutine。

在goroutine中，使用 `select` 调用 `<-ctx.Done()` 判断是否要结束，如果接受到值的话，就可以返回结束goroutine了；如果接收不到，就会继续进行监控。

那么是如何发送结束指令的呢？这就是示例中的 `cancel` 函数啦，它是我们调用`context.WithCancel(parent)`函数生成子Context的时候返回的，第二个返回值就是这个取消函数，它是CancelFunc类型的。我们调用它就可以发出取消指令，然后我们的监控goroutine就会收到信号，就会返回结束。

### Context控制多个goroutine

使用Context控制一个goroutine的例子如上，非常简单，下面我们看看控制多个goroutine的例子，其实也比较简单。

```go
func main() {
	ctx, cancel := context.WithCancel(context.Background())
	go watch(ctx,"【监控1】")
	go watch(ctx,"【监控2】")
	go watch(ctx,"【监控3】")

	time.Sleep(10 * time.Second)
	fmt.Println("可以了，通知监控停止")
	cancel()
	//为了检测监控过是否停止，如果没有监控输出，就表示停止了
	time.Sleep(5 * time.Second)
}

func watch(ctx context.Context, name string) {
	for {
		select {
		case <-ctx.Done():
			fmt.Println(name,"监控退出，停止了...")
			return
		default:
			fmt.Println(name,"goroutine监控中...")
			time.Sleep(2 * time.Second)
		}
	}
}
```

示例中启动了3个监控goroutine进行不断的监控，每一个都使用了 `Context` 进行跟踪，当我们使用 `cancel` 函数通知取消时，这3个goroutine都会被结束。这就是 `Context` 的控制能力，它就像一个控制器一样，按下开关后，所有基于这个 `Context` 或者衍生的子 `Context` 都会收到通知，这时就可以进行清理操作了，最终释放goroutine，这就优雅的解决了goroutine启动后不可控的问题。

> 这里其实还看不出 `context` 相对与 `select + chan` 这种方式的优势来。待后续对 `context`这种方式有跟深入的了解后在添加相关的比较
> 
> 更复杂的场景如何做并发控制呢？比如子协程中开启了新的子协程，或者需要同时控制多个子协程。这种场景下，select+chan的方式就显得力不从心了。
>
> Go 语言提供了 Context 标准库可以解决这类场景的问题，Context 的作用和它的名字很像，上下文，即子协程的下上文。Context 有两个主要的功能：
> 
> - 通知子协程退出（正常退出，超时退出等）；
> - 传递必要的参数。

# Context定义

`context` 的主要数据结构是一种嵌套的结构或者说是单向的继承关系的结构，比如最初的 `context` 是一个小盒子，里面装了一些数据，之后从这个 `context` 继承下来的 `children` 就像在原本的 `context` 中又套上了一个盒子，然后里面装着一些自己的数据。或者说context是一种分层的结构，根据使用场景的不同，每一层context都具备有一些不同的特性，这种层级式的组织也使得context易于扩展，职责清晰。

context 包的核心是 `struct Context`，声明如下：

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    
    Done() <-chan struct{}
    
    Err() error
    
	Value(key interface{}) interface{}
}
```

可以看到 `Context` 是一个 interface，在golang里面，interface是一个使用非常广泛的结构，它可以接纳任何类型。 `Context` 定义很简单，一共4个方法，我们需要能够很好的理解这几个方法

- `Deadline` 是获取设置的截止时间的意思，第一个返回式是截止时间，到了这个时间点，Context会自动发起取消请求；第二个返回值 `ok==false` 时表示没有设置截止时间，如果需要取消的话，需要调用取消函数进行取消。

- `Done` 返回一个只读的 `chan` ，类型为 `struct{}` ，我们在goroutine中，如果该方法返回的 `chan` 可以读取，则意味着 `parent context` 已经发起了取消请求，我们通过 `Done` 方法收到这个信号后，就应该做清理操作，然后退出goroutine，释放资源。之后， `Err` 方法会返回一个错误，告知为什么 `Context` 被取消。

- `Err` 返回取消的错误原因，因为什么 `Context` 被取消。

- `Value` 获取该 `Context` 上绑定的值，是一个键值对，所以要通过一个 `Key` 才可以获取对应的值，这个值一般是线程安全的。


# Context 的实现方法

`Context` 虽然是个接口，但是并不需要使用方实现，golang内置的 `context` 包，已经帮我们实现了2个方法，一般在代码中，开始上下文的时候都是以这两个作为最顶层的parent context，然后再衍生出子 context 。这些 `Context` 对象形成一棵树：当一个 `Context` 对象被取消时，继承自它的所有 `Context` 都会被取消。两个实现如下：

```go
var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

func Background() Context {
	return background
}
func TODO() Context {
	return todo
}
```

- `Background` 主要用于main函数、初始化以及测试代码中，作为 `Context` 这个树结构的最顶层的 `Context` ，也就是根 `Context` ，它不能被取消。

- `TODO` 如果我们不知道该使用什么 `Context` 的时候，可以使用这个，但是实际应用中，暂时还没有使用过这个 `TODO` 。

他们两个本质上都是 `emptyCtx` 结构体类型，是一个不可取消，没有设置截止时间，没有携带任何值的 `Context`。

```go
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}
```

# Context 的继承

有了如上的根 `Context` ，那么是如何衍生更多的子 `Context` 的呢？这就要靠 `context` 包为我们提供的 `With` 系列的函数了。

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) 

func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)

func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)

func WithValue(parent Context, key, val interface{}) Context
```

通过这些函数，就创建了一颗 `Context` 树，树的每个节点都可以有任意多个子节点，节点层级可以有任意多个。

- `WithCancel` 传递一个父 `Context` 作为参数，返回子 `Context` ，以及一个取消函数用来取消 `Context` 。

- `WithDeadline` 和 `WithCancel` 差不多，它会多传递一个截止时间参数，意味着到了这个时间点，会自动取消 `Context` ，当然我们也可以不等到这个时候，可以提前通过取消函数进行取消。

- `WithTimeout` 和 `WithDeadline` 基本上一样，这个表示是超时自动取消，是多少时间后自动取消 `Context` 的意思。

- `WithValue` 和取消 `Context` 无关，它是为了生成一个绑定了一个键值对数据的 `Context` ，这个绑定的数据可以通过 `Context.Value` 方法访问到，这是我们实际用经常要用到的技巧，一般我们想要通过上下文来传递数据时，可以通过这个方法，如我们需要 `tarce` 追踪系统调用栈的时候。

# With 系列函数详解

## WithCancel

`context.WithCancel` 生成了一个 `withCancel` 的实例以及一个 `cancelFuc` ，这个函数就是用来关闭 `ctxWithCancel` 中的 Done channel 函数。

下面来分析下源码实现，首先看看初始化，如下：

```go
// newCancelCtx returns an initialized cancelCtx.
func newCancelCtx(parent Context) cancelCtx {
	return cancelCtx{Context: parent}
}

func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```

`newCancelCtx` 返回一个初始化的 `cancelCtx` ， `cancelCtx` 结构体继承了 `Context` ，实现了 `canceler` 方法：

```go
//*cancelCtx 和 *timerCtx 都实现了canceler接口，实现该接口的类型都可以被直接canceled
type canceler interface {
	cancel(removeFromParent bool, err error)
	Done() <-chan struct{}
}

type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}

func (c *cancelCtx) Done() <-chan struct{} {
	c.mu.Lock()
	if c.done == nil {
		c.done = make(chan struct{})
	}
	d := c.done
	c.mu.Unlock()
	return d
}

func (c *cancelCtx) Err() error {
	c.mu.Lock()
	err := c.err
	c.mu.Unlock()
	return err
}

type stringer interface {
	String() string
}

//核心是关闭c.done
//同时会设置c.err = err, c.children = nil
//依次遍历c.children，每个child分别cancel
//如果设置了removeFromParent，则将c从其parent的children中删除
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done)
	}
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c) // 从此处可以看到 cancelCtx的Context项是一个类似于parent的概念
	}
}
```

可以看到，所有的 `children` 都存在一个 `map` 中； `Done` 方法会返回其中的`done channel`， 而另外的 `cancel` 方法会关闭 Done channel 并且逐层向下遍历，关闭 `children` 的 `channel` ，并且将当前 `canceler` 从 `parent` 中移除。
`WithCancel` 初始化一个 `cancelCtx` 的同时，还执行了 `propagateCancel` 方法，最后返回一个 `cancel function`。

`propagateCancel` 方法定义如下：

```go
// propagateCancel arranges for child to be canceled when parent is.
func propagateCancel(parent Context, child canceler) {
	if parent.Done() == nil {
		return // parent is never canceled
	}
	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
			// parent has already been canceled
			child.cancel(false, p.err)
		} else {
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}
```

`propagateCancel` 的含义就是传递 `cancel`, 由于 `parent` 可能是 `cancel`, `done`, `timeout`, `empty` 中的任意一种，因此:

- 如果 `parent` 已经关闭，则返回，当前 `context` 为 新的 `cancelCtx` 的根

- 从当前传入的 `parent` 开始（包括该parent），向上查找最近的一个可以被 `cancel` 的 `parent` ， 
   - 如果找到的 `parent` 已经被cancel，则将方才传入的 `child` 树给 `cancel` 掉，
   - 否则，将 `child` 节点直接连接为找到的 `parent` 的 `children`中（Context字段不变，即向上的父亲指针不变，但是向下的孩子指针变直接了）；

- 如果没有找到最近的可以被 `cancel` 的 `parent` ，即其上都不可被 `cancel` ，则启动一个goroutine等待传入的 `parent` 终止，则 `cancel` 传入的 `child` 树，或者等待传入的 `child` 终结。

## WithDeadLine

在 `withCancel` 的基础上进行的扩展，如果时间到了之后就进行 `cancel` 的操作，具体的操作流程基本上与 `withCancel` 一致，只不过控制 `cancel` 函数调用的时机是有一个 `timeout` 的 `channel` 所控制的。

# Context 使用原则和技巧

- 不要把 `Context` 放在结构体中，要以参数的方式传递，`parent Context` 一般为 `Background`
- 应该要把 `Context` 作为第一个参数传递给入口请求和出口请求链路上的每一个函数，放在第一位，变量名建议都统一，如 `ctx`。
- 给一个函数方法传递 `Context` 的时候，不要传递 `nil` ，否则在 `tarce` 追踪的时候，就会断了连接
- `Context` 的Value相关方法应该传递必须的数据，不要什么数据都使用这个传递
`Context` 是线程安全的，可以放心的在多个goroutine中传递
- 可以把一个 `Context` 对象传递给任意个数的 gorotuine，对它执行 `cancel` 操作时，所有 goroutine 都会接收到取消信号。

# Context 常用方法实例

## Done

```go
func Stream(ctx context.Context, out chan<- Value) error {

	for {
		v, err := DoSomething(ctx)

		if err != nil {
			return err
		}
		select {
		case <-ctx.Done():

			return ctx.Err()
		case out <- v:
		}
	}
}
```

## WithValue

```go
func main() {
	ctx, cancel := context.WithCancel(context.Background())

	valueCtx := context.WithValue(ctx, key, "add value")

	go watch(valueCtx)
	time.Sleep(10 * time.Second)
	cancel()

	time.Sleep(5 * time.Second)
}

func watch(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			//get value
			fmt.Println(ctx.Value(key), "is cancel")

			return
		default:
			//get value
			fmt.Println(ctx.Value(key), "int goroutine")

			time.Sleep(2 * time.Second)
		}
	}
}
```

## WithTimeout

```go
package main

import (
	"fmt"
	"sync"
	"time"

	"golang.org/x/net/context"
)

var (
	wg sync.WaitGroup
)

func work(ctx context.Context) error {
	defer wg.Done()

	for i := 0; i < 1000; i++ {
		select {
		case <-time.After(2 * time.Second):
			fmt.Println("Doing some work ", i)

		// we received the signal of cancelation in this channel
		case <-ctx.Done():
			fmt.Println("Cancel the context ", i)
			return ctx.Err()
		}
	}
	return nil
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 4*time.Second)
	defer cancel()

	fmt.Println("Hey, I'm going to do some work")

	wg.Add(1)
	go work(ctx)
	wg.Wait()

	fmt.Println("Finished. I'm going home")
}
```

## WithDeadline

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	d := time.Now().Add(1 * time.Second)
	ctx, cancel := context.WithDeadline(context.Background(), d)

	// Even though ctx will be expired, it is good practice to call its
	// cancelation function in any case. Failure to do so may keep the
	// context and its parent alive longer than necessary.
	defer cancel()

	select {
	case <-time.After(2 * time.Second):
		fmt.Println("oversleep")
	case <-ctx.Done():
		fmt.Println(ctx.Err())
	}
}
```

# References

- [Golang Context深入理解](https://juejin.im/post/5a6873fef265da3e317e55b6#heading-5)
		       
- [Go语言实战笔记（二十）| Go Context](https://www.flysnow.org/2017/05/12/go-in-action-go-context.html)
