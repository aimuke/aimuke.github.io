---
title: "How to Gracefully Close Channels"
tags: [go, close, channel]
---

Several days ago, I wrote an article which explains [the channel rules in Go](https://go101.org/article/channel.html) . That article got many votes on reddit and HN, but there are also some criticisms on Go channel design details.

I collected some criticisms on the following designs and rules of Go channels:
1. no easy and universal ways to check whether or not a channel is closed without modifying the status of the channel.
2. closing a closed channel will panic, so it is dangerous to close a channel if the closers don't know whether or not the channel is closed.
3. sending values to a closed channel will panic, so it is dangerous to send values to a channel if the senders don't know whether or not the channel is closed.

The criticisms look reasonable (in fact not). Yes, there is really not a built-in function to check whether or not a channel has been closed.

There is indeed a simple method to check whether or not a channel is closed if you can make sure no values were (and will be) ever sent to the channel. The method has been shown in [the last article](https://go101.org/article/channel-use-cases.html#check-closed-status). Here, for a better coherence, the method is listed in the following example again.

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

As above mentioned, this is not a universal way to check whether a channel is closed.

In fact, even if there is a simple built-in `closed` function to check whether or not a channel has been closed, its usefulness would be very limited, just like the built-in `len` function for checking the current number of values stored in the value buffer of a channel. The reason is the status of the checked channel may have changed just after a call to such functions returns, so that the returned value has already not been able to reflect the latest status of the just checked channel. Although it is okay to stop sending values to a channel `ch` if the call `closed(ch)` returns `true` , it is not safe to close the channel or continue sending values to the channel if the call `closed(ch)` returns `false` .

# The Channel Closing Principle

One general principle of using Go channels is `don't close a channel from the receiver side and don't close a channel if the channel has multiple concurrent senders`. In other words, we should only close a channel in a sender goroutine if the sender is the only sender of the channel.

(Below, we will call the above principle as `channel closing principle`.)

Surely, this is not a universal principle to close channels. The universal principle is `don't close (or send values to) closed channels`. If we can guarantee that no goroutines will close and send values to a non-closed non-nil channel any more, then a goroutine can close the channel safely. However, making such guarantees by a receiver or by one of many senders of a channel usually needs much effort, and often makes code complicated. On the contrary, it is much easy to hold the `channel closing principle` mentioned above.

# Solutions Which Close Channels Rudely
If you would close a channel from the receiver side or in one of the multiple senders of the channel anyway, then you can use [the recover mechanism](https://go101.org/article/control-flows-more.html#panic-recover) to prevent the possible panic from crashing your program. Here is an example (assume the channel element type is `T`).

```go
func SafeClose(ch chan T) (justClosed bool) {
	defer func() {
		if recover() != nil {
			// The return result can be altered
			// in a defer function call.
			justClosed = false
		}
	}()

	// assume ch != nil here.
	close(ch)   // panic if ch is closed
	return true // <=> justClosed = true; return
}
```

This solution obviously breaks the `channel closing principle`.

The same idea can be used for sending values to a potential closed channel.

```go
func SafeSend(ch chan T, value T) (closed bool) {
	defer func() {
		if recover() != nil {
			closed = true
		}
	}()

	ch <- value  // panic if ch is closed
	return false // <=> closed = false; return
}
```

Not only does the rude solution break the `channel closing principle`, and data races might happen in the process.

# Solutions Which Close Channels Politely

Many people prefer using `sync.Once` to close channels:

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

Surely, we can also use `sync.Mutex` to avoid closing a channel multiple times:

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

These ways may be polite, but they may not avoid data races. Currently, Go specification doesn't guarantee that there are no data races happening when a channel close and a channel send operations are executed concurrently. If a `SafeClose` function is called concurrently with a channel send operation to the same channel, data races might happen (though such data races generally don't much harm).

# Solutions Which Close Channels Gracefully

One drawback of the above `SafeSend` function is that its calls can't be used as send operations which follow the `case` keyword in `select` blocks. The other drawback of the above `SafeSend` and `SafeClose` functions is that many people, including me, would think the above solutions by using `panic/recover` and `sync` package are not graceful. Following, some pure-channel solutions without using `panic/recover` and `sync` package will be introduced, for all kinds of situations.

(In the following examples, `sync.WaitGroup` is used to make the examples complete. It may be not always essential to use it in real practice.)

## M receivers, one sender

M receivers, one sender, the sender says "no more sends" by closing the data channel
This is the simplest situation, just let the sender close the data channel when it doesn't want to send more.

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

	wgReceivers := sync.WaitGroup{}
	wgReceivers.Add(NumReceivers)

	// ...
	dataCh := make(chan int)

	// the sender
	go func() {
		for {
			if value := rand.Intn(Max); value == 0 {
				// The only sender can close the
				// channel at any time safely.
				close(dataCh)
				return
			} else {
				dataCh <- value
			}
		}
	}()

	// receivers
	for i := 0; i < NumReceivers; i++ {
		go func() {
			defer wgReceivers.Done()

			// Receive values until dataCh is
			// closed and the value buffer queue
			// of dataCh becomes empty.
			for value := range dataCh {
				log.Println(value)
			}
		}()
	}

	wgReceivers.Wait()
}
```

## One receiver, N senders

One receiver, N senders, the only receiver says "please stop sending more" by closing an additional signal channel

This is a situation a little more complicated than the above one. We can't let the receiver close the data channel to stop data transferring, for doing this will break the `channel closing principle`. But we can let the receiver close an additional signal channel to notify senders to stop sending values.

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
	const NumSenders = 1000

	wgReceivers := sync.WaitGroup{}
	wgReceivers.Add(1)

	// ...
	dataCh := make(chan int)
	stopCh := make(chan struct{})
		// stopCh is an additional signal channel.
		// Its sender is the receiver of channel
		// dataCh, and its receivers are the
		// senders of channel dataCh.

	// senders
	for i := 0; i < NumSenders; i++ {
		go func() {
			for {
				// The try-receive operation is to try
				// to exit the goroutine as early as
				// possible. For this specified example,
				// it is not essential.
				select {
				case <- stopCh:
					return
				default:
				}

				// Even if stopCh is closed, the first
				// branch in the second select may be
				// still not selected for some loops if
				// the send to dataCh is also unblocked.
				// But this is acceptable for this
				// example, so the first select block
				// above can be omitted.
				select {
				case <- stopCh:
					return
				case dataCh <- rand.Intn(Max):
				}
			}
		}()
	}

	// the receiver
	go func() {
		defer wgReceivers.Done()

		for value := range dataCh {
			if value == Max-1 {
				// The receiver of channel dataCh is
				// also the sender of stopCh. It is
				// safe to close the stop channel here.
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

As mentioned in the comments, for the additional signal channel, its sender is the receiver of the data channel. The additional signal channel is closed by its only sender, which holds the `channel closing principle`.

In this example, the channel `dataCh` is never closed. Yes, channels don't have to be closed. **A channel will be eventually garbage collected if no goroutines reference it any more, whether it is closed or not. So the gracefulness of closing a channel here is not to close the channel**.

## M receivers, N senders

M receivers, N senders, any one of them says "let's end the game" by notifying a moderator to close an additional signal channel

This is a the most complicated situation. We can't let any of the receivers and the senders close the data channel. And we can't let any of the receivers close an additional signal channel to notify all senders and receivers to exit the game. Doing either will break the `channel closing principle`. However, we can introduce a moderator role to close the additional signal channel. One trick in the following example is how to use a `try-send` operation to notify the moderator to close the additional signal channel.

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
	const Max = 100000
	const NumReceivers = 10
	const NumSenders = 1000

	wgReceivers := sync.WaitGroup{}
	wgReceivers.Add(NumReceivers)

	// ...
	dataCh := make(chan int)
	stopCh := make(chan struct{})
		// stopCh is an additional signal channel.
		// Its sender is the moderator goroutine shown
		// below, and its receivers are all senders
		// and receivers of dataCh.
	toStop := make(chan string, 1)
		// The channel toStop is used to notify the
		// moderator to close the additional signal
		// channel (stopCh). Its senders are any senders
		// and receivers of dataCh, and its receiver is
		// the moderator goroutine shown below.
		// It must be a buffered channel.

	var stoppedBy string

	// moderator
	go func() {
		stoppedBy = <-toStop
		close(stopCh)
	}()

	// senders
	for i := 0; i < NumSenders; i++ {
		go func(id string) {
			for {
				value := rand.Intn(Max)
				if value == 0 {
					// Here, the try-send operation is
					// to notify the moderator to close
					// the additional signal channel.
					select {
					case toStop <- "sender#" + id:
					default:
					}
					return
				}

				// The try-receive operation here is to
				// try to exit the sender goroutine as
				// early as possible. Try-receive and
				// try-send select blocks are specially
				// optimized by the standard Go
				// compiler, so they are very efficient.
				select {
				case <- stopCh:
					return
				default:
				}

				// Even if stopCh is closed, the first
				// branch in this select block might be
				// still not selected for some loops
				// (and for ever in theory) if the send
				// to dataCh is also non-blocking. If
				// this is unacceptable, then the above
				// try-receive operation is essential.
				select {
				case <- stopCh:
					return
				case dataCh <- value:
				}
			}
		}(strconv.Itoa(i))
	}

	// receivers
	for i := 0; i < NumReceivers; i++ {
		go func(id string) {
			defer wgReceivers.Done()

			for {
				// Same as the sender goroutine, the
				// try-receive operation here is to
				// try to exit the receiver goroutine
				// as early as possible.
				select {
				case <- stopCh:
					return
				default:
				}

				// Even if stopCh is closed, the first
				// branch in this select block might be
				// still not selected for some loops
				// (and forever in theory) if the receive
				// from dataCh is also non-blocking. If
				// this is not acceptable, then the above
				// try-receive operation is essential.
				select {
				case <- stopCh:
					return
				case value := <-dataCh:
					if value == Max-1 {
						// Here, the same trick is
						// used to notify the moderator
						// to close the additional
						// signal channel.
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

In this example, the `channel closing principle` is still held.

Please note that the buffer size (capacity) of channel `toStop` is one. This is to avoid the first notification is missed when it is sent before the moderator goroutine gets ready to receive notification from `toStop`.

We can also set the capacity of the `toStop` channel as the sum number of senders and receivers, then we don't need a try-send `select` block to notify the moderator.

```go
...
toStop := make(chan string, NumReceivers + NumSenders)
...
			value := rand.Intn(Max)
			if value == 0 {
				toStop <- "sender#" + id
				return
			}
...
				if value == Max-1 {
					toStop <- "receiver#" + id
					return
				}
...
```

## third-party close

A variant of the "M receivers, one sender" situation: the close request is made by a third-party goroutine

Sometimes, it is needed that the close signal must be made by a third-party goroutine. For such cases, we can use an extra signal chanel to notify the sender to close the data channel. For example,

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
	closing := make(chan struct{}) // signal channel
	closed := make(chan struct{})
	
	// The stop function can be called
	// multiple times safely.
	stop := func() {
		select {
		case closing<-struct{}{}:
			<-closed
		case <-closed:
		}
	}
	
	// some third-party goroutines
	for i := 0; i < NumThirdParties; i++ {
		go func() {
			r := 1 + rand.Intn(3)
			time.Sleep(time.Duration(r) * time.Second)
			stop()
		}()
	}

	// the sender
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

	// receivers
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

The idea used in the `stop` function is learned from [a comment](https://groups.google.com/forum/#!msg/golang-nuts/lEKehHH7kZY/SRmCtXDZAAAJ) made by Roger Peppe.

## channel must be closed

A variant of the "N sender" situation: the data channel must be closed to tell receivers that data sending is over

In the solutions for the above N-sender situations, to hold the channel closing principle, we avoid closing the data channels. However, sometimes, it is required that the data channels must be closed in the end to let receivers know data sending is over. For such cases, we can translate a N-sender situation to a one-sender situation by using a middle channel. The middle channel has only one sender, so that we can close it instead of closing the original data channel.

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
	const Max = 1000000
	const NumReceivers = 10
	const NumSenders = 1000
	const NumThirdParties = 15

	wgReceivers := sync.WaitGroup{}
	wgReceivers.Add(NumReceivers)

	// ...
	dataCh := make(chan int)     // will never be closed
	middleCh := make(chan int)   // will be closed
	closing := make(chan string) // signal channel
	closed := make(chan struct{})

	var stoppedBy string

	// The stop function can be called
	// multiple times safely.
	stop := func(by string) {
		select {
		case closing <- by:
			<-closed
		case <-closed:
		}
	}
	
	// the middle layer
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
	
	// some third-party goroutines
	for i := 0; i < NumThirdParties; i++ {
		go func(id string) {
			r := 1 + rand.Intn(3)
			time.Sleep(time.Duration(r) * time.Second)
			stop("3rd-party#" + id)
		}(strconv.Itoa(i))
	}

	// senders
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

	// receivers
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

## More situations?

There should be more situation variants, but the above shown ones are the most common and basic ones. By using channels (and other concurrent programming techniques) cleverly, I believe a solution holding the channel closing principle for each situation variant can always be found.

# Conclusion

There are no situations which will force you to break the channel closing principle. If you encounter such a situation, please rethink your design and rewrite you code.

Programming with Go channels is like making art.

> The Go 101 project is hosted on both [Github](https://github.com/go101/go101) and [Gitlab](https://gitlab.com/Go101/go101). Welcome to improve Go 101 articles by submitting corrections for all kinds of mistakes, such as typos, grammar errors, wording inaccuracies, description flaws, code bugs and broken links.
> 
> If you would like to learn some Go details and facts every serveral days, please follow Go 101's official Twitter account: [@go100and1](https://twitter.com/go100and1).
# References

- [原文 - How to Gracefully Close Channels](https://go101.org/article/channel-closing.html)
