---
title: "通过 profiling 定位 golang 性能问题 - 内存篇"
tags: [go, pprof, memory, todo]
---

> 线上性能问题的定位和优化是程序员进阶的必经之路，定位问题的方式有多种多样，常见的有观察线程栈、排查日志和做性能分析。性能分析（profile）作为定位性能问题的大杀器，它可以收集程序执行过程中的具体事件，并且对程序进行抽样统计，从而能更精准的定位问题。本文会以 go 语言的 pprof 工具为例，分享两个线上性能故障排查过程，希望能通过本文使大家对性能分析有更深入的理解。


# cpu 占用 99%

某天早上一到公司就收到了线上 cpu 占用率过高的报警。立即去看监控，发现这个故障主要有下面四个特征：

* cpu idle 基本掉到了 0% ，内存使用量有小幅度增长但不严重；
* 故障是偶发的，不是持续存在的；
* 故障发生时 3 台机器的 cpu 几乎是同时掉底；
* 故障发生后，两个小时左右能恢复正常。

现象如图，上为内存，下为 cpu idle：

![&#x901A;&#x8FC7; profiling &#x5B9A;&#x4F4D; golang &#x6027;&#x80FD;&#x95EE;&#x9898; - &#x5185;&#x5B58;&#x7BC7;](https://static001.infoq.cn/resource/image/56/c7/5662e86f70f17f79fc8747da2051fcc7.png)

检查完监控之后，立即又去检查了一下有没有影响线上业务。看了一下线上接口返回值和延迟，基本上还都能保持正常使用，就算 cpu 占用 99% 时接口延时也只比平常多了几十 ms。由于不影响线上业务，所以没有选择立即回滚，而是决定在线上定位问题（而且前一天后端也确实没有上线新东西）。

所以给线上环境加上了 `pprof` ，等着这个故障自己复现。代码如下：

```go
import _ "net/http/pprof"

func main() {    
    go func() {        
        log.Println(http.ListenAndServe("0.0.0.0:8005", nil))    
    }()    
    // ..... 下面业务代码不用动
}
```

golang 对于 profiling 的支持比较完善，如代码所示，只需要简单的引入 `net/http/pprof` 这个包，然后在 main 函数里启动一个 http server 就相当于给线上服务加上 profiling 了，通过访问 8005 这个 http 端口就可以对程序做采样分析。

服务上开启 pprof 之后，在本地电脑上使用 `go tool pprof` 命令，可以对线上程序发起采样请求，golang pprof 工具会把采样结果绘制成一个漂亮的前端页面供人们排查问题。

等到故障再次复现时，我们首先对 cpu 性能进行采样分析：

```sh
brew install graphviz # 安装 graphviz，只需要安装一次就行了 
go tool pprof -http=:1234 http://your-prd-addr:8005/debug/pprof/profile?seconds=30 
```

打开 terminal，输入上面命令，把命令中的 `your-prd-addr` 改成线上某台机器的地址，然后回车等待 30 秒后，会自动在浏览器中打开一个页面，这个页面包含了刚刚 30 秒内对线上 cpu 占用情况的一个概要分析。点击左上角的 View 选择 Flame graph，会用火焰图（Flame graph）来显示 cpu 的占用情况：

![&#x901A;&#x8FC7; profiling &#x5B9A;&#x4F4D; golang &#x6027;&#x80FD;&#x95EE;&#x9898; - &#x5185;&#x5B58;&#x7BC7;](https://static001.infoq.cn/resource/image/d8/c6/d867aaa1726da943c92e881f3008c0c6.png)

分析此图可以发现，cpu 资源的半壁江山都被 `GetLeadCallRecordByLeadId` 这个函数占用了，这个函数里占用 cpu 最多的又大多是数据库访问相关的函数调用。由于 `GetLeadCallRecordByLeadId` 此函数业务逻辑较为复杂，数据库访问较多，不太好具体排查是哪里出的问题，所以我把这个方向的排查先暂时搁置，把注意力放到了右边那另外半壁江山。

在火焰图的右边，有个让我比较在意的点是 `runtime.gcBgMarkWorker` 函数，这个函数是 golang 垃圾回收相关的函数，用于标记（mark）出所有是垃圾的对象。一般情况下此函数不会占用这么多的 cpu，出现这种情况一般都是内存 gc 问题，但是刚刚的监控上看内存占用只比平常多了几百 M，并没有特别高又是为什么呢？原因是影响 GC 性能的一般都不是内存的占用量，而是对象的数量。举例说明，10 个 100m 的对象和一亿个 10 字节的对象占用内存几乎一样大，但是回收起来一亿个小对象肯定会被 10 个大对象要慢很多。

插一段 golang 垃圾回收的知识，golang 使用“三色标记法”作为垃圾回收算法，是“标记 - 清除法”的一个改进，相比“标记 - 清除法”优点在于它的标记（mark）的过程是并发的，不会 `Stop The World`。但缺点是对于巨量的小对象处理起来比较不擅长，有可能出现垃圾的产生速度比收集的速度还快的情况。`gcMark` 线程占用高很大几率就是对象产生速度大于垃圾回收速度了。

![&#x901A;&#x8FC7; profiling &#x5B9A;&#x4F4D; golang &#x6027;&#x80FD;&#x95EE;&#x9898; - &#x5185;&#x5B58;&#x7BC7;](https://static001.infoq.cn/resource/image/3b/d9/3b23c839845b0a6d54a7d848184be7d9.png)
三色标记法

所以转换方向，又对内存做了一下 profiling：

```sh
go tool pprof http://your-prd-addr:8005/debug/pprof/heap 
```

然后在浏览器里点击左上角 `VIEW->flame graph`，然后点击 `SAMPLE->inuse\_objects`。

这样显示的是当前的对象数量：

![&#x901A;&#x8FC7; profiling &#x5B9A;&#x4F4D; golang &#x6027;&#x80FD;&#x95EE;&#x9898; - &#x5185;&#x5B58;&#x7BC7;](https://static001.infoq.cn/resource/image/87/02/87e425208e255a6f6320e4806cd43e02.png)

可以看到，还是 `GetLeadCallRecordByLeadId` 这个函数的问题，它自己就产生了 1 亿个对象，远超其他函数。所以下一步排查问题的方向确定了是：定位为何此函数产生了如此多的对象。

之后我开始在日志中 `grep ‘/getLeadCallRecord’ lead-platform`. 来一点一点翻，重点看 cpu 掉底那个时刻附近的日志有没有什么异常。果然发现了一条比较异常的日志：

```text
[net/http.HandlerFunc.ServeHTTP/server.go:1947] _com_request_in||traceid=091d682895eda2fsdffsd0cbe3f9a95||spanid=297b2a9sdfsdfsdfb8bf739||hintCode=||hintContent=||method=GET||host=10.88.128.40:8000||uri=/lp-api/v2/leadCallRecord/getLeadCallRecord||params=leadId={"id":123123}||from=10.0.0.0||proto=HTTP/1.0
```

注意看 params 那里， `leadId` 本应该是一个 int，但是前端给传来一个 JSON，推测应该是前一天上线带上去的 bug。但是还有问题解释不清楚，类型传错应该报错，但是为何会产生这么多对象呢？于是我进代码（已简化）里看了看：复制代码

```go
func GetLeadCallRecord(leadId string, bizType int) ([]model.LeadCallRecords, error) {
    sql := "SELECT record.* FROM lead_call_record AS record " +"where record.lead_id  = {{leadId}} and record.biz_type = {{bizType}}"
    conditions := make(map[string]interface{}, 2)
    conditions["leadId"] = leadId
    conditions["bizType"] = bizType
    cond, val, err := builder.NamedQuery(sql, conditions)
```

发现很尴尬的是，这段远古代码里对于 `leadId` 根本没有判断类型，直接用 string 了，前端传什么过来都直接当作 sql 参数了。也不知道为什么 mysql 很抽风的是，虽然 `lead_id` 字段类型是 bigint，在 sql 里条件用 string 类型传参数 WHERE leadId = ‘someString’ 也能查到数据，而且返回的数据量很大。本身 `lead_call_record` 就是千万级别的大表，这个查询一下子返回了几十万条数据。又因为此接口后续的查询都是根据这个这个查询返回的数据进行查询的，所以整个请求一下子就产生了上亿个对象。

由于之前传参都是正确的，所以一直没有触发这个问题，正好前一天前端小姐姐上线需求带上了这个 bug，一波前后端混合双打造成了这次故障。

到此为止就终于定位到了问题所在，而且最一开始的四个现象也能解释的通了：

* cpu idle 基本掉到了 0% ，内存使用量有小幅度增长但不严重；
* 故障是偶发的，不是持续存在的；
* 故障发生时 3 台机器的 cpu 几乎是同时掉底；
* 故障发生后，两个小时左右能恢复正常。

逐条解释一下：

* `GetLeadCallRecordByLeadId` 函数每次在执行时从数据库取回的数据量过大，大量 cpu 时间浪费在反序列化构造对象 和 gc 回收对象上。
* 和前端确认 `/lp-api/v2/leadCallRecord/getLeadCallRecord` 接口并不是所有请求都会传入 json，只在某个页面里才会有这种情况，所以故障是偶发的。
* 因为接口并没有直接挂掉报错，而是执行的很慢，所以应用前面的负载均衡会超时，负载均衡器会把请求打到另一台机器上，结果每次都会导致三台机器同时爆表。
* 虽然申请了上亿个对象，但 golang 的垃圾回收器是真滴靠谱，兢兢业业的回收了两个多小时之后，就把几亿个对象全回收回去了，而且奇迹般的没有影响线上业务。几亿个对象都扛得住，只能说厉害了我的 go。

最后捋一下整个过程：

cpu 占用 99% -> 发现 GC 线程占用率持续异常 -> 怀疑是内存问题 -> 排查对象数量 -> 定位产生对象异常多的接口 -> 定位到某接口 -> 在日志中找到此接口的异常请求 -> 根据异常参数排查代码中的问题 -> 定位到问题

可以发现，有 `pprof` 工具在手的话，整个排查问题的过程都不会懵逼，基本上一直都照着正确的方向一步一步定位到问题根源。这就是用 profiling 的优点所在。

# 内存占用 90%

第二个例子是某天周会上有同学反馈说项目内存占用达到了 15 个 G 之多，看了一下监控现象如下：

* cpu 占用并不高，最低 idle 也有 85%
* 内存占用呈锯齿形持续上升，且速度很快，半个月就从 2G 达到了 15G

如果所示：

![&#x901A;&#x8FC7; profiling &#x5B9A;&#x4F4D; golang &#x6027;&#x80FD;&#x95EE;&#x9898; - &#x5185;&#x5B58;&#x7BC7;](https://static001.infoq.cn/resource/image/4a/62/4a2f9bfa781d58c41c0273313209f662.png)

锯齿是因为昼夜高峰平峰导致的暂时不用管，但持续上涨很明显的是内存泄漏的问题，有对象在持续产生，并且被持续引用着导致释放不掉。于是上了 `pprof` 然后准备等一晚上再排查，让它先泄露一下再看现象会比较明显。

这次重点看内存的 `inuse_space` 图，和 `inuse_objects` 图不同的是，这个图表示的是具体的内存占用而不是对象数，然后 VIEW 类型也选 graph，比火焰图更清晰。

![&#x901A;&#x8FC7; profiling &#x5B9A;&#x4F4D; golang &#x6027;&#x80FD;&#x95EE;&#x9898; - &#x5185;&#x5B58;&#x7BC7;](https://static001.infoq.cn/resource/image/bf/5b/bf8a1fe45ecd0dcbe27b5e1860b60f5b.png)

这个图可以明显的看出来程序中 92% 的对象都是由于 `event.GetInstance` 产生的。然后令人在意的点是这个函数产生的对象都是一个只有 16 个字节的对象（看图上那个 16B）这个是什么原因导致的后面会解释。

先来看这个函数的代码吧：

```go
var (    
    firstActivationEventHandler FirstActivationEventHandler    
    firstOnlineEventHandler FirstOnlineEventHandler
)

func GetInstance(eventType string) Handler {
    if eventType == FirstActivation {
        firstActivationEventHandler.ChildHandler = firstActivationEventHandler
        return firstActivationEventHandler    
    } else if eventType == FirstOnline {
        firstOnlineEventHandler.ChildHandler = firstOnlineEventHandler
        return firstOnlineEventHandler
    }
    // ... 各种类似的判断，略过    return nil}
```

这个是做一个类似单例模式的功能，根据事件类型返回不同的 Handler。但是这个函数有问题的点有两个：

* `firstActivationEventHandler.ChildHandler` 是一个 interface，在给一个 interface 赋值的时候，如果等号右边是一个 struct，会进行值传递，也就意味着每次赋值都会在堆上复制一个此 struct 的副本。（golang 默认都是值传递）
* `firstActivationEventHandler.ChildHandler = firstActivationEventHandler` 是一个自己引用自己循环引用。

两个问题导致了每次 `GetInstance` 函数在被调用的时候，都会复制一份之前的 `firstActivationEventHandler` 在堆上，并且让 `firstActivationEventHandler.ChildHandler` 引用指向到这个副本上。

这就导致人为在内存里创造了一个巨型的链表：

![&#x901A;&#x8FC7; profiling &#x5B9A;&#x4F4D; golang &#x6027;&#x80FD;&#x95EE;&#x9898; - &#x5185;&#x5B58;&#x7BC7;](https://static001.infoq.cn/resource/image/9e/bd/9e0838b1575a9d969dd6a6972f61dfbd.png)

并且这个链表中所有节点都被之后的副本引用着，永远无法被 GC 当作垃圾释放掉。

所以解决这个问题方案也很简单，单例模式只需要在 `init` 函数里初始化一次就够了，没必要在每次 `GetInstance` 的时候做初始化操作：

```go
func init() {
    firstActivationEventHandler.ChildHandler = &firstActivationEventHandler    firstOnlineEventHandler.ChildHandler = &firstOnlineEventHandler
    // ... 略过
}
```

另外，可以深究一下为什么都是一个 `16B` 的对象呢？为什么 `interface` 会复制呢？这里贴一下 golang `runtime` 关于 `interface` 部分的源码：

下面分析 golang 源码，不感兴趣可直接略过。

```go
// interface 底层定义
type iface struct {
    tab  *itab
    data unsafe.Pointer
}

// 空 interface 底层定义

type eface struct {
    _type *_type
    data  unsafe.Pointer
}

// 将某变量转换为 
interfacefunc convT2I(tab *itab, elem unsafe.Pointer) (i iface) {
    t := tab._type
    if raceenabled {
        raceReadObjectPC(t, elem, getcallerpc(), funcPC(convT2I))
    }
    if msanenabled {
        msanread(elem, t.size)
    }
    x := mallocgc(t.size, t, true)
    typedmemmove(t, x, elem)
    i.tab = tab
    i.data = x
    return
}
```

`iface` 这个 `struct` 是 `interface` 在内存中实际的布局。可以看到，在 golang 中定义一个 interface，实际上在内存中是一个 `tab` 指针和一个 `data` 指针，目前的机器都是 64 位的，一个指针占用 8 个字节，两个就是 `16B`。

我们的 `firstActivationEventHandler` 里面只有一个 `ChildHandler interface`，所以整个 `firstActivationEventHandler` 占用 16 个字节也就不奇怪了。

另外看代码第 20 行那里，可以看到每次把变量转为 interface 时是会做一次 `mallocgc(t.size, t, true)` 操作的，这个操作就会在堆上分配一个副本，第 21 行 `typedmemmove(t, x, elem)` 会进行复制，会复制变量到堆上的副本上。这就解释了开头的问题。


# 最高内存使用达到了10G

[原文](https://developer.aliyun.com/article/573743)
> 背景
>
> 阿里云Redis线上在某些任务流中使用redis-port来进行实例之间的数据同步。redis-port是一个MIT协议的开源软件，主要原理是从源实例读取RDB快照文件、解析、然后在目标实例上应用灌数据的写命令。为了限制每个进程的最大内存使用，我们使用cgroup来做隔离，最近线上出现redis-port在同步数据时OOM的情况，最高内存使用达到了10G以上，而实际RDB的大小只有4.5GB左右。

分析内存使用要是光撸代码还是比较困难的，总要借助一些工具。Golang pprof是Golang官方的profiling工具，非常强大，使用也比较方便。

我们在程序中嵌入如下几行代码，

```go
import _ "net/http/pprof"

func main() {
    // ...
    go func() {
        http.ListenAndServe("0.0.0.0:8899", nil)
    }()
    // ...
}
```

在浏览器中输入 `http://ip:8899/debug/pprof/` 可以看到一个汇总页面，

```
/debug/pprof/

profiles:
0    block
32    goroutine
552    heap
0    mutex
51    threadcreate

full goroutine stack dump
```

其中 `heap` 项是我们需要关注的信息，

```
heap profile: 96: 1582948832 [21847: 15682528480] @ heap/1048576
91: 1527472128 [246: 4129210368] @ 0x471d87 0x471611 0x4718cd 0x4689bf 0x50deb9 0x50d7ac 0x75893b 0x45d801
#    0x471d86    bytes.makeSlice+0x76                            /usr/local/go/src/bytes/buffer.go:231
#    0x471610    bytes.(*Buffer).grow+0x140                        /usr/local/go/src/bytes/buffer.go:133
#    0x4718cc    bytes.(*Buffer).Write+0xdc                        /usr/local/go/src/bytes/buffer.go:163
#    0x4689be    io.(*multiWriter).Write+0x8e                        /usr/local/go/src/io/multi.go:60
#    0x50deb8    github.com/CodisLabs/redis-port/pkg/rdb.createValueDump+0x198        go_workspace/src/github.com/CodisLabs/redis-port/pkg/rdb/loader.go:194
#    0x50d7ab    github.com/CodisLabs/redis-port/pkg/rdb.(*Loader).NextBinEntry+0x28b    go_workspace/src/github.com/CodisLabs/redis-port/pkg/rdb/loader.go:176
#    0x75893a    main.newRDBLoader.func1+0x23a                        go_workspace/src/github.com/CodisLabs/redis-port/cmd/utils.go:733
......
```

包括一些汇总信息，和各个go routine的内存开销，不过这里除了第一行信息比较直观，其他的信息太离散。可以看到当前使用的堆内存是 1.58GB，总共分配过 15.6GB。

```
heap profile: 96(inused_objects): 1582948832(inused_bytes) [21847(allocated_objects): 15682528480(allocted_bytes)] @ heap/1048576
```

更有用的信息我们需要借助 `go tool pprof` 来进行分析，

```sh
go tool pprof -alloc_space/-inuse_space http://ip:8899/debug/pprof/heap
```

这里有两个选项，`-alloc_space` 和 `-inuse_space`，从名字应该能看出二者的区别，不过条件允许的话，我们优先使用 `-inuse_space` 来分析，因为直接分析导致问题的现场比分析历史数据肯定要直观的多，一个函数 `alloc_space` 多不一定就代表它会导致进程的RSS高，因为我们比较幸运可以在线下复现这个OOM的场景，所以直接用 `-inuse_space` 。

这个命令进入后，是一个类似gdb的交互式界面，输入top命令可以前10大的内存分配，flat是堆栈中当前层的inuse内存值，cum是堆栈中本层级的累计inuse内存值（包括调用的函数的inuse内存值，上面的层级），

```sh
(pprof) top
Showing nodes accounting for 3.73GB, 99.78% of 3.74GB total
Dropped 5 nodes (cum <= 0.02GB)
Showing top 10 nodes out of 16
      flat  flat%   sum%        cum   cum%
    3.70GB 98.94% 98.94%     3.70GB 98.94%  bytes.makeSlice /usr/local/go/src/bytes/buffer.go
    0.03GB  0.83% 99.78%     0.03GB  0.83%  main.(*cmdRestore).Main /usr/local/go/src/bufio/bufio.go
         0     0% 99.78%     3.70GB 98.94%  bytes.(*Buffer).Write /usr/local/go/src/bytes/buffer.go
         0     0% 99.78%     3.70GB 98.94%  bytes.(*Buffer).grow /usr/local/go/src/bytes/buffer.go
         0     0% 99.78%     3.70GB 98.94%  github.com/CodisLabs/redis-port/pkg/rdb.(*Loader).NextBinEntry go_workspace/src/github.com/CodisLabs/redis-port/pkg/rdb/loader.go
         0     0% 99.78%     3.70GB 98.94%  github.com/CodisLabs/redis-port/pkg/rdb.(*rdbReader).Read go_workspace/src/github.com/CodisLabs/redis-port/pkg/rdb/reader.go
         0     0% 99.78%     3.70GB 98.94%  github.com/CodisLabs/redis-port/pkg/rdb.(*rdbReader).ReadBytes go_workspace/src/github.com/CodisLabs/redis-port/pkg/rdb/reader.go
         0     0% 99.78%     3.70GB 98.94%  github.com/CodisLabs/redis-port/pkg/rdb.(*rdbReader).ReadString go_workspace/src/github.com/CodisLabs/redis-port/pkg/rdb/reader.go
         0     0% 99.78%     3.70GB 98.94%  github.com/CodisLabs/redis-port/pkg/rdb.(*rdbReader).readFull go_workspace/src/github.com/CodisLabs/redis-port/pkg/rdb/reader.go
         0     0% 99.78%     3.70GB 98.94%  github.com/CodisLabs/redis-port/pkg/rdb.(*rdbReader).readObjectValue go_workspace/src/github.com/CodisLabs/redis-port/pkg/rdb/reader.go
```

可以看到大部分内存都是 `bytes.makeSlice` 产生的（flat 98.94%），不过这是一个标准库函数，再撸撸代码，往下看可以看到redis-port实现的函数 `(*Loader).NextBinEntry` ，这里推荐使用 `list` 命令，

```sh
(pprof) list NextBinEntry
Total: 3.74GB
ROUTINE ======================== github.com/CodisLabs/redis-port/pkg/rdb.(*Loader).NextBinEntry in go_workspace/src/github.com/CodisLabs/redis-port/pkg/rdb/loader.go
         0     3.70GB (flat, cum) 98.94% of Total
         .          .    137:           default:
         .          .    138:                   key, err := l.ReadString()
         .          .    139:                   if err != nil {
         .          .    140:                           return nil, err
         .          .    141:                   }
         .     3.70GB    142:                   val, err := l.readObjectValue(t)
         .          .    143:                   if err != nil {
         .          .    144:                           return nil, err
         .          .    145:                   }
         .          .    146:                   entry.DB = l.db
         .          .    147:                   entry.Key = key
```

可以直接看到这个函数在哪一行代码产生了多少的内存！不过如果是在可以方便导出文件的测试环境，推荐使用命令，

```sh
go tool pprof -inuse_space -cum -svg http://ip:8899/debug/pprof/heap > heap_inuse.svg
image.png
```

![内存占用](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/db9379265bd07d9aeff6ef2f3a5b93da.png)

这个可以得到前后调用关系的调用栈图，同时还包括每一层的 `inuse` 内存大小，文件名，函数，到下一层的内存大小，分析起来简直不能再顺手。

最后定位原因就比较简单了，redis-port在解析RDB时，是按key为粒度来处理的，遇到大key时，value可能有好几个GB，然后redis-port直接使用了标准库 `bytes.Buffer` 来存储解析出来的 value（对于redis hash来说是field，value对），Buffer在空间不够的时候会自己grow，策略是当前capacity 2倍的增长速度，避免频繁内存分配，看看标准库的代码（go 1.9）

```go
// grow grows the buffer to guarantee space for n more bytes.
// It returns the index where bytes should be written.
// If the buffer can't grow it will panic with ErrTooLarge.
func (b *Buffer) grow(n int) int {
......
    } else {
        // Not enough space anywhere, we need to allocate.
        buf := makeSlice(2*cap(b.buf) + n)
        copy(buf, b.buf[b.off:])
        b.buf = buf
    }
......
}
```

Buffer在空间不够时，申请一个当前空间2倍的byte数组，然后把老的copy到这里，这个峰值内存就是3倍的开销，如果value大小5GB，读到4GB空间不够，那么创建一个8GB的新buffer，那么峰值就是12GB了，此外Buffer的初始大小是64字节，在增长到4GB的过程中也会创建很多的临时byte数组，gc不及时也是额外的内存开销，所以4.5GB的RDB，在有大key的情况下，峰值内存用到15GB也就可以理解了。

## Fix
这个问题的根本原因还是按key处理一次读的value太大，在碰到hash这种复杂数据类型时，其实我们可以分而治之，读了一部分value后，比如16MB就生成一个子hash，避免Buffer grow产生太大的临时对象。

此外，解析RDB时，受限于RDB的格式，只能单个go routine处理，但是回放时，是可以由多个go routine来并发处理多个子hash，写到目标实例的。每个子hash处理完，又可以被gc及时的清理掉。同时并发度上去了，同步的速度也有所提升（主要受限于目标Redis，因为Redis是单线程处理请求）。

![youhuaqian1.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/32b3a136f219354b5580c62e899bf348.png)

![youhuahou1.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/1e0c8337d86a923c090e0840d1ca2fa5.png)

最后，做个简单的对比，可以看到优化后redis-port的RSS峰值为2.6GB，和之前相比降低了80%。

# 经验总结

在做内存问题相关的 profiling 时：

* 若 gc 相关函数占用异常，可重点排查对象数量
* 解决速度问题（CPU 占用）时，关注对象数量（ `--inuse/alloc_objects` ）指标
* 解决内存占用问题时，关注分配空间（ `--inuse/alloc_space` ）指标

inuse 代表当前时刻的内存情况，alloc 代表从从程序启动到当前时刻累计的内存情况，一般情况下看 inuse 指标更重要一些，但某些时候两张图对比着看也能有些意外发现。

在日常 golang 编码时：

* 参数类型要检查，尤其是 sql 参数要检查（低级错误）
* 传递 struct 尽量使用指针，减少复制和内存占用消耗（尤其对于赋值给 interface，会分配到堆上，额外增加 gc 消耗）
* 尽量不使用循环引用，除非逻辑真的需要
* 能在初始化中做的事就不要放到每次调用的时候做

**本文转载自公众号滴滴技术（ID：`didi_tech`）。**

# References

- [通过 profiling 定位 golang 性能问题 - 内存篇](https://www.infoq.cn/article/f69uvzJUOmq276HBp1Qb), [张威虎](https://www.infoq.cn/profile/1671484/publish)

- [记一次Golang内存分析——基于go pprof](https://developer.aliyun.com/article/573743), [夏周tony](https://developer.aliyun.com/profile/ahvjbqxfkylde?spm=a2c6h.12873639.0.0.55dede55bjD8Z4)
