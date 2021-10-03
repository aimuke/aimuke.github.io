---
title: "如何优雅地关闭 channel"
tags: [go, channel, close]
---

几天前，我写了一篇介绍 Go 中管道有关的规则的文章。 这篇文章在 reddit 和 HN 得到很多投票。

我收集了一些关于 Go 中管道的设计和规则的意见：

- 没有简单易行的方法去检查管道是否没有通过改变它的状态来关闭。
- 关闭一个已经关闭的管道会触发 `panic` ，所以，关闭者不知道管道是否关闭仍去关闭它，这是一个危险的行为。
- 发送数据到一个关闭的管道会触发 `panic` , 所以，发送者不知道管道是否关闭仍去发送消息给它，这是一个危险的行为。

这些意见看起来很合理 （其实并不是）。然而 Go 中没有一个內建函数去检测管道是否已经被关闭。

如果你能保证没有任何数据发送到管道，这有一个简单的方法去检查管道是否被关闭 （这个方法在这篇文章的其他例子中会被经常用到）:

```go
package main

import "fmt"

type T int

func IsClosed(ch <-chan T) bool {
    select {
    case <-ch:
        return true
    default:
    }

    return false
}

func main() {
    c := make(chan T)
    fmt.Println(IsClosed(c)) // false
    close(c)
    fmt.Println(IsClosed(c)) // true
}
```

如上所述，这不是一个普遍的方式去检测一个管道是否关闭。

然而，这 `closed(chan T) bool` 是一个很简单的检查一个通道是否被关闭的方法，但它的用处还是很有限的，就像内建函数 `len` 获取存储在通道值缓存中的当前值数量。 理由是在调用这个函数后通道的状态可能会发生改变，所以返回值并不能反应出刚检查完通道的最后状态。即使给一个调用 `close(ch)` 函数返回 `true` 的通道 `ch` 停止发送值这是可行的，但给一个调用 `chose(ch)` 函数返回 `false` 的通道关闭或继续发送值是不安全的。

# 通道关闭原则

一般原则上使用通道是不允许接收方关闭通道和 不能关闭一个有多个并发发送者的通道。 换而言之， 你只能在发送方的 `goroutine` 中关闭只有该发送方的通道。

（下面， 我们将上述原则称之为通道关闭原则。）

当然，对于关闭通道来说这不是一个普遍的原则。 普遍原则是不发送数据给（或关闭）一个关闭状态通道。 如果任意一个 goroutine 能保证没有 goroutine 会再发送数据给（或关闭）一个没有关闭不为空的通道， 这样的话 goruoutine 可以安全地关闭通道。然而，从通道接受方或者多个发送方之一实现这样的保证需要花费很大的工夫，而且通常会把代码写得很复杂。相反，遵守上文所提到的 `通道关闭原则` 更加简单。

# 粗暴关闭通道的解决方案
无论如何，如果你要从接收方或通道的多个发送方之一中关闭通道， 那么你可以使用 `recover` 机制去预防可能的 `panic` 破坏你的程序。这里有一个例子 （假设通道的元素类型是 `T`）。

```go
func SafeClose(ch chan T) (justClosed bool) {
    defer func() {
        if recover() != nil {
            // 返回值可以被修改
            // 在一个延时函数的调用中。
            justClosed = false
        }
    }()

    // 假设这里 ch != nil 。
    close(ch)   // 如果 ch 已经被关闭将会引发 panic
    return true // <=> justClosed = true; return
}
```

这解决方案明显违反了 `通道关闭原则`。

同样的思想可以用于将数据发送到一个潜在的关闭通道：

```go
func SafeSend(ch chan T, value T) (closed bool) {
    defer func() {
        if recover() != nil {
            closed = true
        }
    }()

    ch <- value  // 如果 ch 已经被关闭将会引发 panic
    return false // <=> closed = false; return
}
```

# 优雅关闭通道的解决方案

很多人更喜欢使用 `sync.Once` 去关闭通道：

```go
type MyChannel struct {
    C    chan T
    once sync.Once
}

func NewMyChannel() *MyChannel {
    return &MyChannel{C: make(chan T)}
}

func (mc *MyChannel) SafeClose() {
    mc.once.Do(func() {
        close(mc.C)
    })
}
```

当然，我们也会使用 `sync.Mutex` 去避免多次关闭一个通道：

```go
type MyChannel struct {
    C      chan T
    closed bool
    mutex  sync.Mutex
}

func NewMyChannel() *MyChannel {
    return &MyChannel{C: make(chan T)}
}

func (mc *MyChannel) SafeClose() {
    mc.mutex.Lock()
    defer mc.mutex.Unlock()
    if !mc.closed {
        close(mc.C)
        mc.closed = true
    }
}

func (mc *MyChannel) IsClosed() bool {
    mc.mutex.Lock()
    defer mc.mutex.Unlock()
    return mc.closed
}
```

我们应该能猜到 Go 不支持內建 `SafeSend` 和 `SafeClose` 函数的理由是从接收方或多发送方关闭通道这种行为是不推荐的。 Go 甚至禁止关闭只读通道。

# 优雅关闭通道的方案

上文提到的 `SafeSend` 函数的其中一个缺陷是它不能在 `select` 代码块中 `case` 关键词后面作为一个发送行为被调用。 上文提到的 `SafeSend` 和 `SafeClose` 函数的另一个缺陷是很多人，包括我，都会觉得上文使用 `panic/recover` 和 `sync` 包的解决方案并不优雅。 下面，将会介绍一些不使用 `panic/recover` 和 `sync` 包的纯通道解决方案，这些方案适用于所有情况。

（在接下来的例子中，`sync.WaitGroup` 会被用于完成例子。 它在实践中并不是必要的。）

## M 个接受者，1 个发送者

M 个接受者，1 个发送者， 发送者通过关闭数据通道说 「不要再发送了」
这是最简单的情况，只需让发送者在不想再发送数据的时候关闭数据通道：

```go
package main

import (
    "time"
    "math/rand"
    "sync"
    "log"
)

func main() {
    rand.Seed(time.Now().UnixNano())
    log.SetFlags(0)

    // ...
    const MaxRandomNumber = 100000
    const NumReceivers = 100

    wgReceivers := sync.WaitGroup{}
    wgReceivers.Add(NumReceivers)

    // ...
    dataCh := make(chan int, 100)

    // 发送者
    go func() {
        for {
            if value := rand.Intn(MaxRandomNumber); value == 0 {
                // 唯一的发送者可以安全地关闭通道。
                close(dataCh)
                return
            } else {
                dataCh <- value
            }
        }
    }()

    // 接收者
    for i := 0; i < NumReceivers; i++ {
        go func() {
            defer wgReceivers.Done()

            // 接收数据直到 dataCh 被关闭或者
            // dataCh 的数据缓存队列是空的。
            for value := range dataCh {
                log.Println(value)
            }
        }()
    }

    wgReceivers.Wait()
}
```

# 1 个接收者，N 个发送者

1 个接收者，N 个发送者，接收者通过关闭一个信号通道说 「请不要再发送数据了」
这个情况比上面的要复杂一点。 我们不能让接收者关闭数据通道，for 不然就会违反了 通道关闭原则。但是我们可以让接收者关闭一个信号通道去通知接收者停止发送数据：

```go
package main

import (
    "time"
    "math/rand"
    "sync"
    "log"
)

func main() {
    rand.Seed(time.Now().UnixNano())
    log.SetFlags(0)

    // ...
    const MaxRandomNumber = 100000
    const NumSenders = 1000

    wgReceivers := sync.WaitGroup{}
    wgReceivers.Add(1)

    // ...
    dataCh := make(chan int, 100)
    stopCh := make(chan struct{})
        // stopCh 是一个信号通道。
        // 它的发送者是 dataCh 的接收者。
        // 它的接收者是 dataCh 的发送者。

    // 发送者
    for i := 0; i < NumSenders; i++ {
        go func() {
            for {
                // 第一个 select 是为了尽可能早的尝试退出 goroutine。
                //  事实上，在这个特殊的例子中，这不是必要的，所以它能省略。
                select {
                case <- stopCh:
                    return
                default:
                }

                // 即使 stopCh 已经关闭，如果发送给 dataCh 没有阻塞，那么在第二个 select 中第一个分支可能会在一些循环中不会执行。
                // 但是在这里例子中是可接受的， 所以上面的第一个 select 代码块可以被省略。
                select {
                case <- stopCh:
                    return
                case dataCh <- rand.Intn(MaxRandomNumber):
                }
            }
        }()
    }

    // 接收者
    go func() {
        defer wgReceivers.Done()

        for value := range dataCh {
            if value == MaxRandomNumber-1 {
                //  dataCh 通道的接收者也是 stopCh 通道的发送者。
                // 在这里关闭停止通道是安全的。.
                close(stopCh)
                return
            }

            log.Println(value)
        }
    }()

    // ...
    wgReceivers.Wait()
}
```

正如注释所说，对于信号通道来说，它的发送者是数据通道的接收者。 根据 通道关闭原则，信号通道只能由它唯一的发送者关闭。

在这里例子中， 通道 `dataCh` 不曾被关闭过。 是的，通道没有必要关闭。 `如果一个通道不会再有 goroutine 去使用它，它最终会被垃圾回收， 不管它是否被关闭。 所以在这里优雅的关闭通道就是不要去关闭通道。`

## M 个接收者，N 个发送者

M 个接收者，N 个发送者， 其中，任意一个通过通知一个主持人去关闭一个信号通道说「让我们结束这场游戏吧」

这是最复杂的情况。 我们不能让任意一个接收者或者发送者关闭数据通道。我们也不能让任意一个接收者通过关闭信号通道去通知所有的发送者和接收者退出这场游戏。 实现任意一个都会违反 通道关闭原则。 然而，我们可以创建一个主持人角色去关闭信号通道。 这个例子中的一个技巧就是怎样通知主持人关闭信号通道：

```go
package main

import (
    "time"
    "math/rand"
    "sync"
    "log"
    "strconv"
)

func main() {
    rand.Seed(time.Now().UnixNano())
    log.SetFlags(0)

    // ...
    const MaxRandomNumber = 100000
    const NumReceivers = 10
    const NumSenders = 1000

    wgReceivers := sync.WaitGroup{}
    wgReceivers.Add(NumReceivers)

    // ...
    dataCh := make(chan int, 100)
    stopCh := make(chan struct{})
        // stopCh 是一个信号通道。
        // 它的发送者是下面的主持人 goroutine。
        // 它的接收者是 dataCh的所有发送者和接收者。
    toStop := make(chan string, 1)
        // toStop 通道通常用来通知主持人去关闭信号通道( stopCh )。
        // 它的发送者是 dataCh的任意发送者和接收者。
        // 它的接收者是下面的主持人 goroutine

    var stoppedBy string

    // 主持人
    go func() {
        stoppedBy = <-toStop
        close(stopCh)
    }()

    // 发送者
    for i := 0; i < NumSenders; i++ {
        go func(id string) {
            for {
                value := rand.Intn(MaxRandomNumber)
                if value == 0 {
                    // 这里，一个用于通知主持人关闭信号通道的技巧。
                    select {
                    case toStop <- "sender#" + id:
                    default:
                    }
                    return
                }

                // 这里的第一个 select 是为了尽可能早的尝试退出 goroutine。
                // 标准编译器对这种try-receive 和 try-send 的 代码块是特殊优化过的，所以效率很高
                select {
                case <- stopCh:
                    return
                default:
                }

                // 即使stopCh已关闭，如果这个select代码块
                // 中第二个分支的发送操作是非阻塞的，则第一个
                // 分支仍很有可能在若干个循环步内依然不会被选
                // 中。如果这是不可接受的，则上面的第一个尝试
                // 接收操作代码块是必需的。
                select {
                case <- stopCh:
                    return
                case dataCh <- value:
                }
            }
        }(strconv.Itoa(i))
    }

    // 接收者
    for i := 0; i < NumReceivers; i++ {
        go func(id string) {
            defer wgReceivers.Done()

            for {
                // 和发送者 goroutine 一样， 这里的第一个 select 是为了尽可能早的尝试退出 这个 goroutine。
                // 这个 select 代码块有 1 个发送行为 的 case 和 一个将作为 Go 编译器的 try-send 操作进行特别优化的默认分支。
                select {
                case <- stopCh:
                    return
                default:
                }

                // 即使stopCh已关闭，如果这个select代码块
                // 中第二个分支的接收操作是非阻塞的，则第一个
                // 分支仍很有可能在若干个循环步内依然不会被选
                // 中。如果这是不可接受的，则上面尝试接收操作
                // 代码块是必需的。
                select {
                case <- stopCh:
                    return
                case value := <-dataCh:
                    if value == MaxRandomNumber-1 {
                        // 同样的技巧用于通知主持人去关闭信号通道。
                        select {
                        case toStop <- "receiver#" + id:
                        default:
                        }
                        return
                    }

                    log.Println(value)
                }
            }
        }(strconv.Itoa(i))
    }

    // ...
    wgReceivers.Wait()
    log.Println("stopped by", stoppedBy)
}
```

这个例子仍然遵守通道关闭原则。

请注意， `toStop` 通道的缓存大小（容量）是 `1`。 这是为了避免第一个通知在主持人 goroutine 准备接收从 `toStop` 来的通知之前发送的情况下丢失 。

我们也可以设置 `toStop` 通道的容量是发送者和接收者的数量和， 这样我们就不需要一个 try-send `select` 代码块去通知主持人了。

```go
...
toStop := make(chan string, NumReceivers + NumSenders)
...
        value := rand.Intn(MaxRandomNumber)
                if value == 0 {
                toStop <- "sender#" + id
                return
        }
...
        if value == MaxRandomNumber-1 {
                toStop <- "receiver#" + id
                return
        }
...
```

## 第三方关闭

情形四：“M个接收者和一个发送者”情形的一个变种：用来传输数据的通道的关闭请求由第三方发出
有时，数据通道（`dataCh`）的关闭请求需要由某个第三方协程发出。对于这种情形，我们可以使用一个额外的信号通道来通知唯一的发送者关闭数据通道（`dataCh`）。

```go
package main

import (
	"time"
	"math/rand"
	"sync"
	"log"
)

func main() {
	rand.Seed(time.Now().UnixNano())
	log.SetFlags(0)

	// ...
	const Max = 100000
	const NumReceivers = 100
	const NumThirdParties = 15

	wgReceivers := sync.WaitGroup{}
	wgReceivers.Add(NumReceivers)

	// ...
	dataCh := make(chan int)
	closing := make(chan struct{}) // 信号通道
	closed := make(chan struct{})
	
	// 此stop函数可以被安全地多次调用。
	stop := func() {
		select {
		case closing<-struct{}{}:
			<-closed
		case <-closed:
		}
	}
	
	// 一些第三方协程
	for i := 0; i < NumThirdParties; i++ {
		go func() {
			r := 1 + rand.Intn(3)
			time.Sleep(time.Duration(r) * time.Second)
			stop()
		}()
	}

	// 发送者
	go func() {
		defer func() {
			close(closed)
			close(dataCh)
		}()

		for {
			select{
			case <-closing: return
			default:
			}

			select{
			case <-closing: return
			case dataCh <- rand.Intn(Max):
			}
		}
	}()

	// 接收者
	for i := 0; i < NumReceivers; i++ {
		go func() {
			defer wgReceivers.Done()

			for value := range dataCh {
				log.Println(value)
			}
		}()
	}

	wgReceivers.Wait()
}
```

上述代码中的 `stop` 函数中使用的技巧偷自Roger Peppe在此贴中的 [一个留言](https://groups.google.com/forum/#!msg/golang-nuts/lEKehHH7kZY/SRmCtXDZAAAJ)。

## 必须关闭channel

情形五：“N个发送者”的一个变种：用来传输数据的通道必须被关闭以通知各个接收者数据发送已经结束了
在上面的提到的“N个发送者”情形中，为了遵守通道关闭原则，我们避免了关闭数据通道（`dataCh`）。 但是有时候，数据通道（`dataCh`）必须被关闭以通知各个接收者数据发送已经结束。 对于这种“N个发送者”情形，我们可以使用一个中间通道将它们转化为“一个发送者”情形，然后继续使用上一节介绍的技巧来关闭此中间通道，从而避免了关闭原始的 `dataCh` 数据通道。

```
package main

import (
	"time"
	"math/rand"
	"sync"
	"log"
	"strconv"
)

func main() {
	rand.Seed(time.Now().UnixNano())
	log.SetFlags(0)

	// ...
	const Max = 1000000
	const NumReceivers = 10
	const NumSenders = 1000
	const NumThirdParties = 15

	wgReceivers := sync.WaitGroup{}
	wgReceivers.Add(NumReceivers)

	// ...
	dataCh := make(chan int)   // 仍就不会被关闭
	middleCh := make(chan int) // 将被关闭
	closing := make(chan string)
	closed := make(chan struct{})

	var stoppedBy string

	stop := func(by string) {
		select {
		case closing <- by:
			<-closed
		case <-closed:
		}
	}
	
	// 中间层
	go func() {
		exit := func(v int, needSend bool) {
			close(closed)
			if needSend {
				dataCh <- v
			}
			close(dataCh)
		}

		for {
			select {
			case stoppedBy = <-closing:
				exit(0, false)
				return
			case v := <- middleCh:
				select {
				case stoppedBy = <-closing:
					exit(v, true)
					return
				case dataCh <- v:
				}
			}
		}
	}()
	
	// 一些第三方协程
	for i := 0; i < NumThirdParties; i++ {
		go func(id string) {
			r := 1 + rand.Intn(3)
			time.Sleep(time.Duration(r) * time.Second)
			stop("3rd-party#" + id)
		}(strconv.Itoa(i))
	}

	// 发送者
	for i := 0; i < NumSenders; i++ {
		go func(id string) {
			for {
				value := rand.Intn(Max)
				if value == 0 {
					stop("sender#" + id)
					return
				}

				select {
				case <- closed:
					return
				default:
				}

				select {
				case <- closed:
					return
				case middleCh <- value:
				}
			}
		}(strconv.Itoa(i))
	}

	// 接收者
	for range [NumReceivers]struct{}{} {
		go func() {
			defer wgReceivers.Done()

			for value := range dataCh {
				log.Println(value)
			}
		}()
	}

	// ...
	wgReceivers.Wait()
	log.Println("stopped by", stoppedBy)
}
```


## 更多解决方案

基于上面的三种情况，会有更多变体。例如，基于最复杂情况的一个变体可能需要接受者读取缓存数据通道中剩余的所有数据.。这会很容易处理并且本文不会涉及它。

即使上面的 3 种情况不能覆盖 Go 通道使用中的所有情况，但它们是最基础的。 实践中大部分情况都可以分为这 3 种情况。

# 结论
没有任何情况会迫使你违反通道关闭原则。 如果你遇到这样的一个情况，请你重新想想你的设计并重写你的代码。

使用 Go 通道编程就像在制作一个艺术品。

# References

- [原文 - 如何优雅地关闭 channel](https://gfw.go101.org/article/channel-closing.html)
- [原文 - 如何优雅地关闭 channel](https://go101.org/article/channel-closing.html)
