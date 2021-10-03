---
title: "Go按行读取数据的坑"
tags: [go, reader, readline]
---

首先从一个日志分析的Go程序说起，基本功能就是一行一行读取数据并处理。代码大体是这样的：

```go
func main() {
    scanner := bufio.NewScanner(os.Stdin)
    for scanner.Scan() {
        line := scanner.Text()
        if err := deal(line); err != nil {
            log.Println(line)
        }
    }
}
```

因为数据是用hadoop直接 `cat` 出来的，通过管道传输到日志处理程序，所以这里输入源是 `stdin` 。

实际处理的时候发现一个比较奇怪的现象，每个日志文件的总行数差别不大，但是有的文件处理时间明显比其他短，并且还会能看到这样的日志信息：`cat: Unable to write to output stream.`。这个日志不是Go里面的，确认是hadoop报了这个错误，起初以为是hadoop在某些情况下可能会出现这个错误，因为出现这个错误就意味着某些文件没有处理完程序就退出了。这个是不能容忍的。于是便向运维那边报bug。

运维那边承认这个错误是hadoop报的，但是出现这个错误一般是因为stream断了，比如如果有head操作就会出现这个错误：

```sh
$ hdfs dfs -cat  0.log | head -1
*****log contents*****
cat: Unable to write to output stream.
```
# 问题定位

为了证明是后面处理程序的问题而不是hadoop的问题，运维那边做了这样一个操作：

1. 将hadoop中的文件下载到本地
2. 直接用UNIX的cat命令将数据重定向到处理程序
3. 查看cat和处理程序的返回值

```sh
$ cat 0.log | go_program
$ echo ${PIPESTATUS[@]}
141 0
```

`cat` 非 0，后面的程序返回是 `0`，UNIX基本程序一般是不会有问题的，看来是后面的Go程序有问题。开始检查验证。

首先要确认的问题有两个：

- Go程序真的没有处理完数据就退出了吗？
- 如果异常退出了那么是在处理哪一行的时候退出了？

查询这个问题也是比较容易的，将处理过的每一行日志输出并打印行号，这样就知道处理到哪一行了，也就能知道在哪一行出错。

修改程序后重新执行，确实是Go程序提前退出了，并且没处理的下一行唯一的**特殊之处是这一行超级长**。难道Go不能处理很长的一行？这看起来不可能发生，但是还要验证一下。

首先是重新review代码，按理说程序出错之后会有错误日志的，为啥Go程序不但没报错还正常退出了？首先是review数据处理代码，没发现问题，然后就开始怀疑是 `bufio.Scanner` 的问题，重新看了一下Go文档，官网有一个example是这么写的：

```go
scanner := bufio.NewScanner(os.Stdin)
for scanner.Scan() {
    fmt.Println(scanner.Text()) // Println will add back the final '\n'
}
if err := scanner.Err(); err != nil {
    fmt.Fprintln(os.Stderr, "reading standard input:", err)
}
```

原来 `scanner` 还有一个 `Err()` 方法，官方的说法是 **Err returns the first non-EOF error that was encountered by the Scanner.**，也就是说如果 `scanner.Scan()` 如果出错的话错误信息是要通过 `Err()` 方法才能得到的，我的go程序将这个 `Err` 忽略了，出错后退出for循环就直接退出程序了，到底是有什么错误？代码补充完整之后看到这样的错误：`bufio.Scanner: token too long`。

到目前能确认的确实是Go程序问题，在读数据的时候出现错误，上面的错误是什么意思？翻Go代码。

```sh
~/go$ grep -n ` find -name "*.go"` -e "bufio.Scanner: token too long"
./src/pkg/bufio/scan.go:61:     ErrTooLong         = errors.New("bufio.Scanner: token too long")
```

确实有这个错误，继续看 `scan.go`，看看在哪报 `ErrTooLong` 这个错误：

```sh
~/go$ grep -n ` find -name "*.go"` -e "ErrTooLong"
./src/pkg/bufio/scan.go:61:     ErrTooLong         = errors.New("bufio.Scanner: token too long")
./src/pkg/bufio/scan.go:147:                            s.setErr(ErrTooLong)
./src/pkg/bufio/scan_test.go:244:       if err != ErrTooLong {
./src/pkg/bufio/scan_test.go:245:               t.Fatalf("expected ErrTooLong; got %s", err)
```

`scan.go` 147 行出现的错误，看具体代码：

```go
144                  // Is the buffer full? If so, resize.
145                  if s.end == len(s.buf) {
146                          if len(s.buf) >= s.maxTokenSize {
147                                  s.setErr(ErrTooLong)
148                                  return false
149                          }
150                          newSize := len(s.buf) * 2
151                          if newSize > s.maxTokenSize {
152                                  newSize = s.maxTokenSize
153                          }
154                          newBuf := make([]byte, newSize)
155                          copy(newBuf, s.buf[s.start:s.end])
156                          s.buf = newBuf
157                          s.end -= s.start
158                          s.start = 0
159                          continue
160                  }
```

看了一下 `Scanner` 模块的代码，原来是这样的： `Scanner` 在初始化的时候有设置一个 `maxTokenSize` ，这个值默认是 `MaxScanTokenSize = 64 * 1024` ，当一行的长度大于 `64*1024` 即 `65536` 之后，就会出现 `ErrTooLong` 错误。这样就能解释之前为什么长行处理报错的问题了。

# 解决方案

毕竟日志一行可能不止65536这么长，问题还是要解决的，有两种方案：

- 调大maxTokenSize的值
- 换用其他的函数

对于方案1，仔细看了一下 `Scanner` ，并没有发现有接口可以修改这个值，除非修改Go源码并重新编译Go，这个太折腾。在官方文档里面发现这样一段话：

> Scanning stops unrecoverably at EOF, the first I/O error, or a token too large to fit in the buffer. When a scan stops, the reader may have advanced arbitrarily far past the last token. Programs that need more control over error handling or large tokens, or must run sequential scans on a reader, should use bufio.Reader instead.

它告诉我们如果 `token` 太大(行太长)的时候，要使用 `bufio.Reader`。这里是第一个坑，怪当初用的时候没仔细看文档，没仔细研读Go源码。从这里我们可以得到一个教训： **除非能确定行长度不超过65536，否则不要使用`bufio.Scanner`！**

## ReadLine

那接下来老老实实用 `bufio.Reader` 吧。那 `bufio.Reader` 有没有类似 `bufio.Scanner` 的问题呢？

看了一下 `bufio.Reader` 代码，这个也是有缓冲区大小限制的，并且默认缓冲区大小是4096，还好有一个函数 `NewReaderSize` 可以调整这个缓冲区大小。

还是之前的需求，如何按行去读取数据呢？`bufio.Reader`提供了一个 `ReadLine()` 函数，文档上是这么说的：

> ReadLine is a low-level line-reading primitive. Most callers should use ReadBytes('\n') or ReadString('\n') instead or use a Scanner.

意思是这个函数比较底层，建议使用 `ReadBytes` 或 `ReadString` 或者 `Scanner` 。继续看文档说明：

> ReadLine tries to return a single line, not including the end-of-line bytes. If the line was too long for the buffer then isPrefix is set and the beginning of the line is returned. The rest of the line will be returned from future calls. isPrefix will be false when returning the last fragment of the line. The returned buffer is only valid until the next call to ReadLine. ReadLine either returns a non-nil line or it returns an error, never both.

从这里我们能看出来设置缓冲区的作用了， `ReadLine` 会尽量去读取并返回完整的一行，但是如果行太长缓冲区满了的话，就不会返回完整的一行而是返回缓冲区里面的内容，并且会设置 `isPrefix` 为 true。这时候需要继续调用 `ReadLine` 直到将完整一行读完。然后外层调用程序需要将这些块拼起来才能组成完整的行。不仅要处理 `isPrefix` 还要处理前缀，麻烦！除非我们主动设置一个非常大的缓冲，但是前提是你必须知道最长行的长度，在大多数情况下这个是无法预先知道的。怪不得建议我们使用 `ReadBytes` 或 `ReadString` 或者 `Scanner` 。

## ReadString

之前讨论了， `Scanner` 在行太长的时候是有问题的， `ReadBytes` 和 `ReadString` 原理上是一样的，这里我们以 `ReadString` 为例看看这个函数。 `ReadString` 原理很简单，使用也很方便，基本用法是指定一个分隔符，比如我们如果读取一行的话就指定分隔符为 `\n`，这样 `ReadString` 就去不断去读数据，直到发现分隔符 `\n` 或者出错为止。官网是这么说的：

> ReadString reads until the first occurrence of delim in the input, returning a string containing the data up to and including the delimiter. If ReadString encounters an error before finding a delimiter, it returns the data read before the error and the error itself (often io.EOF). ReadString returns err != nil if and only if the returned data does not end in delim. For simple uses, a Scanner may be more convenient.

(ps: 这里又在推荐Scanner...)

`ReadString` 有没有缓冲区大小限制呢？这个其实也还是有的，不过，它在代码里面就将这个缓冲区的问题处理好了，也就是说能保证返回的一行肯定是完整的。最起码用起来比 `ReadLine` 函数方便。

那么用 `ReadString` 去按行读取有没有坑呢？答案是肯定的。

```go
package main

import (
    "bufio"
    "fmt"
    "io"
    "strings"
)

func main() {
    s := "a\nb\nc"
    reader := bufio.NewReader(strings.NewReader(s))
    for {
        line, err := reader.ReadString('\n')
        if err != nil {
            if err == io.EOF {
                break
            }
            panic(err)
        }
        fmt.Printf("%#v\n", line)
    }
}
```

上面这段代码的输出是：

```
"a\n"
"b\n"
```

为什么c没有输出？这里算是一个坑。之前讨论过，按行读取的话 `ReadString` 函数需要以 `\n` 作为分割，上面那种特殊情况当数据末尾没有 `\n` 的时候，直到 `EOF` 还没有分隔符 `\n` ，这时候返回 `EOF` 错误，但是line里面还是有数据的，如果不处理的话就会漏掉最后一行。简单修改一下：

```go
package main

import (
    "bufio"
    "fmt"
    "io"
    "strings"
)

func main() {
    s := "a\nb\nc"
    reader := bufio.NewReader(strings.NewReader(s))
    for {
        line, err := reader.ReadString('\n')
        if err != nil {
            if err == io.EOF {
                fmt.Printf("%#v\n", line)
                break
            }
            panic(err)
        }
        fmt.Printf("%#v\n", line)
    }
}
```

这样执行会输出：

```
"a\n"
"b\n"
"c"
```

这样处理的时候也要当心，因为最后一行后面没有 `\n`。

相比之下，python 处理要简单和清晰很多，有生成器，直接 `for` 循环遍历即可：

```python
import StringIO

s = "a\nb\nc"

sio = StringIO.StringIO(s)

for line in sio:
    print line,
```

# 转发小结

- Scanner: 缓冲区固定 65535, 超出缓冲区大小时输错
- ReadLine: 也有缓冲区（reader 的缓冲区，默认为4096），超出缓冲区时，isPrefix 为true，多次才能读取到完整的一行
- ReadString：可以通过 `ReadString('\n')` 的方式获取到完整的一行信息，需要自己处理换行符 `"\n"`, `"\r\n"`

# References

- [原文 Go按行读取数据的坑](https://github.com/ma6174/blog/issues/10), [Ma Weiwei](https://github.com/ma6174)
