---
title: "bufio — 缓存IO"
tags: [go, bufio, reader]
---

`bufio` 包实现了缓存IO。它包装了 `io.Reader` 和 `io.Writer` 对象，创建了另外的 `Reader` 和 `Writer` 对象，它们也实现了 `io.Reader` 和 `io.Writer` 接口，不过它们是有缓存的。该包同时为文本I/O提供了一些便利操作。

# Reader 类型和方法

`bufio.Reader` 结构包装了一个 `io.Reader` 对象，提供缓存功能，同时实现了 `io.Reader` 接口。

`Reader` 结构没有任何导出的字段，结构定义如下：

```go
type Reader struct {
	buf          []byte        // 缓存
	rd           io.Reader    // 底层的io.Reader
	// r:从buf中读走的字节（偏移）；w:buf中填充内容的偏移；
	// w - r 是buf中可被读的长度（缓存数据的大小），也是Buffered()方法的返回值
	r, w         int
	err          error        // 读过程中遇到的错误
	lastByte     int        // 最后一次读到的字节（ReadByte/UnreadByte)
	lastRuneSize int        // 最后一次读到的Rune的大小 (ReadRune/UnreadRune)
}
```

# 实例化

`bufio` 包提供了两个实例化 `bufio.Reader` 对象的函数： `NewReader` 和 `NewReaderSize` `。其中，NewReader` 函数是调用 `NewReaderSize` 函数实现的：

```go
func NewReader(rd io.Reader) *Reader {
	// 默认缓存大小：defaultBufSize=4096
	return NewReaderSize(rd, defaultBufSize)
}
```
我们看一下 `NewReaderSize` 的源码：

```go
func NewReaderSize(rd io.Reader, size int) *Reader {
	// 已经是bufio.Reader类型，且缓存大小不小于 size，则直接返回
	b, ok := rd.(*Reader)
	if ok && len(b.buf) >= size {
		return b
	}
	// 缓存大小不会小于 minReadBufferSize （16字节）
	if size < minReadBufferSize {
		size = minReadBufferSize
	}
	// 构造一个bufio.Reader实例
	return &Reader{
		buf:          make([]byte, size),
		rd:           rd,
		lastByte:     -1,
		lastRuneSize: -1,
	}
}
```

#  ReadSlice、ReadBytes、ReadString 和 ReadLine 方法

之所以将这几个方法放在一起，是因为他们有着类似的行为。事实上，后三个方法最终都是调用 `ReadSlice` 来实现的。所以，我们先来看看 `ReadSlice` 方法。(感觉这一段直接看源码较好)

## ReadSlice
`ReadSlice` 方法签名如下：

```go
func (b *Reader) ReadSlice(delim byte) (line []byte, err error)
```

`ReadSlice` 从输入中读取，直到遇到第一个界定符（delim）为止，返回一个指向缓存中字节的 `slice` ，在下次调用读操作（read）时，这些字节会无效。举例说明：

```go
reader := bufio.NewReader(strings.NewReader("http://studygolang.com. \nIt is the home of gophers"))
line, _ := reader.ReadSlice('\n')
fmt.Printf("the line:%s\n", line)
// 这里可以换上任意的 bufio 的 Read/Write 操作
n, _ := reader.ReadSlice('\n')
fmt.Printf("the line:%s\n", line)
fmt.Println(string(n))
```

输出：

```
the line:http://studygolang.com. 

the line:It is the home of gophers
It is the home of gophers
```

从结果可以看出，第一次 `ReadSlice` 的结果（line），在第二次调用读操作后，内容发生了变化。也就是说， `ReadSlice` 返回的 `[]byte` 是指向 `Reader` 中的 `buffer` ，而不是 copy 一份返回。正因为 `ReadSlice` 返回的数据会被下次的 I/O 操作重写，因此许多的客户端会选择使用 `ReadBytes` 或者 `ReadString` 来代替。读者可以将上面代码中的 `ReadSlice` 改为 `ReadBytes` 或 `ReadString` ，看看结果有什么不同。

注意，这里的界定符可以是任意的字符，可以将上面代码中的'`\n`'改为'`m`'试试。同时，返回的结果是包含界定符本身的，上例中，输出结果有一空行就是'`\n`'本身(line携带一个'`\n`', `printf` 又追加了一个'`\n`')。

如果 `ReadSlice` 在找到界定符之前遇到了 `error` ，它就会返回缓存中所有的数据和错误本身（经常是 `io.EOF` ）。如果在找到界定符之前缓存已经满了， `ReadSlice` 会返回 `bufio.ErrBufferFull` 错误。当且仅当返回的结果（line）没有以界定符结束的时候， `ReadSlice` 返回 `err != nil`，也就是说，如果 `ReadSlice` 返回的结果 line 不是以界定符 delim 结尾，那么返回的 `err` 也一定不等于 `nil` （可能是bufio.ErrBufferFull或io.EOF）。 例子代码：

```go
reader := bufio.NewReaderSize(strings.NewReader("http://studygolang.com"),16)
line, err := reader.ReadSlice('\n')
fmt.Printf("line:%s\terror:%s\n", line, err)
line, err = reader.ReadSlice('\n')
fmt.Printf("line:%s\terror:%s\n", line, err)
```
输出：

```
line:http://studygola    error:bufio: buffer full
line:ng.com    error:EOF
```
`ReadBytes` 方法签名如下：

```go
func (b *Reader) ReadBytes(delim byte) (line []byte, err error)
```

该方法的参数和返回值类型与 `ReadSlice` 都一样。 `ReadBytes` 从输入中读取直到遇到界定符（delim）为止，返回的 `slice` 包含了从当前到界定符的内容 （包括界定符）。如果 `ReadBytes` 在遇到界定符之前就捕获到一个错误，它会返回遇到错误之前已经读取的数据，和这个捕获到的错误（经常是 io.EOF）。跟  `ReadSlice` 一样，如果 `ReadBytes` 返回的结果 line 不是以界定符 delim 结尾，那么返回的 `err` 也一定不等于 `nil` （可能是bufio.ErrBufferFull 或 io.EOF）。

从这个说明可以看出， `ReadBytes` 和 `ReadSlice` 功能和用法都很像，那他们有什么不同呢？

在讲解 `ReadSlice` 时说到，它返回的 `[]byte` 是指向 `Reader` 中的 `buffer` ，而不是 `copy` 一份返回，也正因为如此，通常我们会使用 `ReadBytes` 或 `ReadString` 。很显然， `ReadBytes` 返回的 `[]byte` 不会是指向 `Reader` 中的 `buffer` ，通过查看源码可以证实这一点。

还是上面的例子，我们将 `ReadSlice` 改为 `ReadBytes` ：

```go
reader := bufio.NewReader(strings.NewReader("http://studygolang.com. \nIt is the home of gophers"))
line, _ := reader.ReadBytes('\n')
fmt.Printf("the line:%s\n", line)
// 这里可以换上任意的 bufio 的 Read/Write 操作
n, _ := reader.ReadBytes('\n')
fmt.Printf("the line:%s\n", line)
fmt.Println(string(n))
```

输出：

```
the line:http://studygolang.com. 

the line:http://studygolang.com. 

It is the home of gophers
```

## ReadString

看一下该方法的源码：

```go
func (b *Reader) ReadString(delim byte) (line string, err error) {
	bytes, err := b.ReadBytes(delim)
	return string(bytes), err
}
```

它调用了 `ReadBytes` 方法，并将结果的 `[]byte` 转为 `string` 类型。

## ReadLine

```go
func (b *Reader) ReadLine() (line []byte, isPrefix bool, err error)
```
`ReadLine` 是一个底层的原始行读取命令。许多调用者或许会使用 `ReadBytes('\n')` 或者 `ReadString`('\n') 来代替这个方法。

`ReadLine` 尝试返回单独的行，不包括行尾的换行符。如果一行大于缓存， `isPrefix` 会被设置为 true，同时返回该行的开始部分（等于缓存大小的部分）。该行剩余的部分就会在下次调用的时候返回。当下次调用返回该行剩余部分时， `isPrefix` 将会是 `false` 。跟 `ReadSlice` 一样，返回的 `line` 只是 `buffer` 的引用，在下次执行IO操作时， `line` 会无效。可以将 `ReadSlice` 中的例子该为 `ReadLine` 试试。

注意，返回值中，要么 line 不是 nil，要么 err 非 nil，两者不会同时非 nil。

`ReadLine` 返回的文本不会包含行结尾（"`\r\n"`或者"`\n`"）。如果输入中没有行尾标识符，不会返回任何指示或者错误。

从上面的讲解中，我们知道，读取一行，通常会选择 `ReadBytes` 或 `ReadString` 。不过，正常人的思维，应该用 `ReadLine` ，只是不明白为啥 `ReadLine` 的实现不是通过 `ReadBytes` ，然后清除掉行尾的 `\n`（或 `\r\n`），它现在的实现，用不好会出现意想不到的问题，比如丢数据。个人建议可以这么实现读取一行：

```go
line, err := reader.ReadBytes('\n')
line = bytes.TrimRight(line, "\r\n")
```

这样既读取了一行，也去掉了行尾结束符（当然，如果你希望留下行尾结束符，只用ReadBytes即可）。

## Peek 方法

从方法的名称可以猜到，该方法只是“窥探”一下 `Reader` 中没有读取的 `n` 个字节。好比栈数据结构中的取栈顶元素，但不出栈。

方法的签名如下：

```go
func (b *Reader) Peek(n int) ([]byte, error)
```

同上面介绍的 `ReadSlice` 一样，返回的 `[]byte` 只是 `buffer` 中的引用，在下次IO操作后会无效，可见该方法（以及 `ReadSlice` 这样的，返回 `buffer` 引用的方法）对多 `goroutine` 是不安全的，也就是在多并发环境下，不能依赖其结果。

我们通过例子来证明一下：

```go
package main

import (
	"bufio"
	"fmt"
	"strings"
	"time"
)

func main() {
	reader := bufio.NewReaderSize(strings.NewReader("http://studygolang.com.\t It is the home of gophers"), 14)
	go Peek(reader)
	go reader.ReadBytes('\t')
	time.Sleep(1e8)
}

func Peek(reader *bufio.Reader) {
	line, _ := reader.Peek(14)
	fmt.Printf("%s\n", line)
	// time.Sleep(1)
	fmt.Printf("%s\n", line)
}
```

输出：

```
http://studygo
http://studygo
```

输出结果和预期的一致。然而，这是由于目前的 goroutine 调度方式导致的结果。如果我们将例子中注释掉的 `time.Sleep(1)` 取消注释（这样调度其他 goroutine 执行），再次运行，得到的结果为：

```
http://studygo
ng.com.     It is
```

另外， `Reader` 的 `Peek` 方法如果返回的 `[]byte` 长度小于 `n`，这时返回的 `err != nil` ，用于解释为啥会小于 `n`。如果 `n` 大于 reader 的 buffer 长度，err 会是 ErrBufferFull。

## 其他方法

`Reader` 的其他方法都是实现了 io 包中的接口，它们的使用方法在io包中都有介绍，在此不赘述。

这些方法包括：

```go
func (b *Reader) Read(p []byte) (n int, err error)
func (b *Reader) ReadByte() (c byte, err error)
func (b *Reader) ReadRune() (r rune, size int, err error)
func (b *Reader) UnreadByte() error
func (b *Reader) UnreadRune() error
func (b *Reader) WriteTo(w io.Writer) (n int64, err error)
```

你应该知道它们都是哪个接口的方法吧。

#　References

- [原文 bufio 缓存](https://books.studygolang.com/The-Golang-Standard-Library-by-Example/chapter01/01.4.html)
