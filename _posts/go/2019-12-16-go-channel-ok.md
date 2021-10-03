---
title: "golang reflect判断channel是否关闭"
tags: [go, close, chan]
---

**rfyiamcool 2018年8月22日**

# 前言:

如何判断golang的 `channel` 是否关闭，我想玩go的同学都知道，`data, ok := <- chan`，当 `ok` 不是 `true` 的时候，说明是 `channel` 关闭了。 那么问题来了， `channel` 关闭了，我们是否可以立马获取到 `channel` 被关闭的状态？我想这个问题不少人没有去想吧？ 

为什么有这样的问题？  来自我的一个bug，我期初认为 `close` 了一个 `channel` ，消费端的goroutine自然是可以拿到 `channel` 的关闭状态。然而事实并不是这样的。 只有当 `channel` 无数据，且 `channel` 被 `close` 了，才会返回 `ok=false`。  所以，只要有堆积，就不会返回关闭状态。导致我的服务花时间来消费堆积，才会退出。

测试channel的关闭状态

```go
package main

import (
    "fmt"
)

func main() {
    c := make(chan int, 10)
    c <- 1
    c <- 2
    c <- 3
    close(c)
    for {
        i,  ok := <-c
        fmt.Println(ok)
        if !ok {
            fmt.Println("channel closed!")
            break
        }
        fmt.Println(i)
    }
}
```

我们发现已经把 `channel` 关闭了，只要有堆积的数据，那么 `ok` 就不为 `false` ，不为关闭的状态。


# go runtime channel源码分析

首先我们来分析下go `runtime/chan.go` 的相关源码，记得先前写过一篇golang channel 实现的源码分析，有兴趣的朋友可以翻翻。 这次翻 channel 源码主要探究下 `close chan` 过程及怎么查看 `channel` 是否关闭？

下面是 `channel` 的 `hchan` 主数据结构， `closed` 字段就是标明是否退出的标识。

```go
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
        ...
}
```

下面是关闭 `channel` 的函数，修改了 `closed` 字段为 `1` ,  `1` 为退出。

```go
//go:linkname reflect_chanclose reflect.chanclose
func reflect_chanclose(c *hchan) {
	closechan(c)
}

func closechan(c *hchan) {
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}
        ....
	c.closed = 1
        ...
}
```

下面是 `channel` 的 `recv` 消费者方法，也就是 `data, ok := <- chan` 。`if c.closed != 0 && c.qcount == 0` 只有当 `closed` 为 `1` 并且 堆积为 `0` 的时候，才会返回 `false` 。 一句话， `channel` 已经 `closed` ，并且没有堆积任务，才会返回关闭 `channel` 的状态。


```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
        ....
	lock(&c.lock)

	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(unsafe.Pointer(c))
		}
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}

	if sg := c.sendq.dequeue(); sg != nil {
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

	if c.qcount > 0 {
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		c.qcount--
		unlock(&c.lock)
		return true, true
	}
   ...
}
```

`channel` 代码里没有找到一个查询 channel 关闭的方法。

# 解决方法

那么如何在channel堆积的情况下，得知channel已经关闭了 ？

## 第一种方法

可以直接读取 `channel` 结构 `hchan` 的 `closed` 字段，但问题 `chan.go` 没有开放这样的api，所以我们要用 `reflect` 这个黑科技了。  （不推荐大家用 `reflect` 的方法，因为看起来太黑科技了）


```go
import (
    "unsafe"
    "reflect"
)

func isChanClosed(ch interface{}) bool {
    if reflect.TypeOf(ch).Kind() != reflect.Chan {
        panic("only channels!")
    }
    cptr := *(*uintptr)(unsafe.Pointer(
        unsafe.Pointer(uintptr(unsafe.Pointer(&ch)) + unsafe.Sizeof(uint(0))),
    ))

    // this function will return true if chan.closed > 0
    // see hchan on https://github.com/golang/go/blob/master/src/runtime/chan.go
    // type hchan struct {
    // qcount   uint           // total data in the queue
    // dataqsiz uint           // size of the circular queue
    // buf      unsafe.Pointer // points to an array of dataqsiz elements
    // elemsize uint16
    // closed   uint32
    // **

    cptr += unsafe.Sizeof(uint(0))*2
    cptr += unsafe.Sizeof(unsafe.Pointer(uintptr(0)))
    cptr += unsafe.Sizeof(uint16(0))
    return *(*uint32)(unsafe.Pointer(cptr)) > 0
}
```

## 第二种方法

配合一个 `context` 或者一个变量来做。就拿 `context` 来说，那么 `select` 不仅可以读取数据 `chan` ，且同时监听 `<- context.Done()` , 当`context.Done()` 有事件，直接退出就 `ok` 了。


```go
    ...
	ctx, cancel := context.WithCancel(context.Background())
	close(c)
	cancel()
exit:
	for {
		select {
		case data, ok := <-c:
			fmt.Println(data, ok)

		case <-ctx.Done():
			break exit
		}
	}
	 ...
```

# 总结:

这个问题肯定不是 `channel` 的问题了，只能说我对 `channel` 理解还是不够，还是要继续钻研golang runtime源码。没了，希望这篇文章对大家有用。

# References

- [原文 - golang reflect判断channel是否关闭](http://xiaorui.cc/2018/08/22/golang-reflect%e5%88%a4%e6%96%adchannel%e6%98%af%e5%90%a6%e5%85%b3%e9%97%ad/)
