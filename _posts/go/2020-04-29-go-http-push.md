---
title: "Golang-长连接-状态推送"
tags: [go, http, connection, "长连接"]
---

**前言**：扫码登录功能自微信提出后，越来越多的被应用于各个web与app。这两天公司要做一个扫码登录功能，在leader的技术支持帮助下（基本都靠leader排坑），终于将服务搭建起来，并且支持上万并发。

# 长连接选择

决定做扫码登录功能之后，在网上查看了很多的相关资料。对于扫码登录的实现方式有很多，淘宝用的是轮询，微信用长连接，QQ用轮询……。方式虽多，但目前看来大体分为两种，1:轮询，2:长连接。（两种方式各有利弊吧，我研究不深，优缺点就不赘述了）

在和leader讨论之后选择了用长连接的方式。所以对长连接的实现方式调研了很多：

## 微信长连接

通过动态加载script的方式实现。

![script](https://segmentfault.com/img/bVbm5vx?w=1023&h=27)

这种方式好在没有跨域问题。

## websocket长连接

在PC端与服务端搭起一条长连接后，服务端主动不断地向PC端推送状态。这应该是最完美的做法了。

## 长连接

PC端向服务端发送请求，服务端并不立即响应，而是hold住，等到用户扫码之后再响应这个请求，响应后连接断开。

![长连接](https://segmentfault.com/img/bVbm5IB?w=948&h=24)

为什么不采用 `websocket` 呢？因为当时比较急、而对于 `websocket` 的使用比较陌生，所以没有使用。不过我现在这种做法在资源使用上比 `websocket` 低很多。

# 接口设计

流程大体如下：
1. 打开PC界面的时候向服务端发送请求并建立长连接。服务端hold住连接不返回
2. 使用APP成功确认后
3. pc端登陆成功跳转页面

分析得出我们的服务只需要两个接口即可
- 与PC建立长连接的接口
- 接收APP端数据并将数据发送给前端的接口

再细想可将这两个接口抽象为：
- PC获取状态接口： `get`
- APP设置状态接口： `set`

# 具体实现

长连接的根本原理：连接请求后，服务端利用 `channel` 阻塞住。等到 `channel` 中有 `value` 后，将 `value` 响应

## Router

```go
func Router(){
    http.HandleFunc("/status/get", Get)
    http.HandleFunc("/status/set", Set)
}
```

## GET

每一条连接需要有一个 `KEY` 作标识，不然 APP 设置的状态不知道该发给那台PC。每一条连接即一个 `channel` 

```go
var Status map[string](chan string) = make(map[string](chan string))

func Get(w http.ResponseWriter, r *http.Request){
    ...        //接收key的操作
    key = ...  //PC在请求接口时带着的key
    Status[key] = make(chan string)    //不需要缓冲区
    value := <-Status[key]
    ResponseJson(w, 0, "success", value)    //自己封的响应JSON方法
}
```

## SET

APP扫码后可以得到二维码中的 `KEY` ，同时将想给PC发送的 `VALUE` 一起发送给服务端

```go
func Set(w http.ResponseWriter, r *http.Request){
    ...        
    key = ...
    value = ...    //向PC传递的值
    Status[key] <- value
}
```

这就是实现的最基本原理。接下来我们一点点实现其他的功能。

# 其他功能

## 超时

从网上找了很多资料，大部分都说这种方式

```go
srv := &http.Server{  
    ReadTimeout: 5 * time.Second,
    WriteTimeout: 10 * time.Second,
}
log.Println(srv.ListenAndServe())
```

这种方式确实是设置读超时与写超时。但（亲测）这种超时方式并不友善，假如现在 `WriteTimeout` 是 `10s` ，PC端请求过来之后，长连接建立。PC处于 `pending` 状态，并且服务端被 `channel` 阻塞住。 `10s` 之后，由于超时连接失效（并没有断，我也不了解其中原理）。PC并不知道连接断了，依然处于 `pending` 状态，服务端的这个 `goroutine` 依然被阻塞在这里。这个时候我调用 `set` 接口，第一次调用没用反应，但第二次调用PC端就能成功接收 `value` 。

![timeout](https://segmentfault.com/img/bVbm6bs?w=849&h=271)

从图可以看出，我设置的 `WriteTimeout` 为 `10s` ，但这条长连接即使 `15s` 依然能收到成功响应。（ps：我调用了两次 `set` 接口，第一次没有反应）

研究后决定不使用这种方式设置超时，采用接口内部定时的方式实现超时返回

```go
select {
    case <-`Timer`:
        utils.ResponseJson(w, -1, "timeout", nil)
    case value := <-statusChan:
        utils.ResponseJson(w, 0, "success", value)
    }
```

`Timer` 即为定时器。刚开始 `Timer` 是这样定义的

```go
Timer := time.After(60 * time.Second)
```

`60s` 后 `Timer` 会自动返回一个值，这时上面的通道就开了，响应 `timeout` ,但这样做有一个弊端，这个定时器一旦创建就必须等待 `60s` ，并且我没想到办法提前将定时器关了。如果这个长连接刚建立后 `5s` 就被响应，那么这个定时器就要多存在 `55s` 。这样对资源是一种浪费，并不合理。

这里选用了 `context` 作为定时器

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Duration(Timeout)*time.Second)
defer cancel()
select {
    case <-ctx.Done():
        utils.ResponseJson(w, -1, "timeout", nil)
    case result := <-Status[key]:
        utils.ResponseJson(w, 0, "success", result)
}
```

`ctx` 在初始化的时候就设置了超时时间 `time.Duration(Timeout)*time.Second`, 超时之后 `ctx.Done()` 返回完成，起到定时作用。如果没有 `cancel()` 则会有一样的问题。原因如下

> [参考地址](https://blog.csdn.net/liangzhiyang/java/article/details/65633960)
>
> 但是对于 `WithTimeout` (或者 `WithDeadline` ) 有两种情况
> 1. 一种是发生超时了，这个时候 `cancel` 会自动调用，资源被释放
> 2. 另一种没有发生超时，也就是 `slowOperation` 结束的时候，这个时候需要咱们主动调用 `cancel` ；但是即使没有调用，在过期时间到了的时候还是会调用 `cancel` ，释放资源
> 
> **所以： `cancel` 即使不主动调用，也不影响资源的最终释放，但是提前主动调用，可以尽快的释放，避免等待过期时间之间的浪费；**
> **建议还是按照官方的说明使用，养成良好的习惯，在调用 `WithTimeout` 之后 `defer cancel()`**

`context` 对比 `time` 包。提供了手动关闭定时器的方法 `cancel()`,只要 `get` 请求结束，都会去关闭定时器，这样可以避免资源浪费（一定程度避免内存泄漏）。

即使golang[官方文档](https://golang.org/pkg/context/)中，也推荐 `defer cancel()` 这样做，其文档中写到：

> Even though ctx will be expired, it is good practice to call its cancellation function in any case. Failure to do so may keep the  context and its parent alive longer than necessary.
> 
> 即使ctx会在到期时关闭，但在任何场景手动调用cancel都是很好的做法。这样超时功能就实现了

## 多机支持

服务如果只部署在一台机器上，万一机器跪了，那就全跪了。所以我们的服务必须同时部署在多个机器上工作。即使其中一台挂了，也不影响服务使用。

![HA](https://segmentfault.com/img/bVbm6xp?w=1341&h=497)

在项目初期讨论的时候leader给出了两种方案

- 如图使用 `redis` 做多机调度。
- 使用 `zookeeper` 将消息发送给多机

因为现在是用 `redis` 做的，只讲述下 `redis` 的实现。（但依赖 `redis` 并不是很好，多机的负载均衡还要依赖其他工具。 `zookeeper` 能够解决这个问题，之后会将 `redis` 换成 `zookeeper` ）

首先我们要明确多机的难点在哪？我们有两个接口，`get`、`set`。`get`是给前端建立长连接用的。`set`是后端设置状态用的。

假设有两台机器A、B。若前端的请求发送到A机器上，即A机器与前端连接，此时后端调用 `set` 接口，如果调用的是A机器的 `set` 接口，那是最好，长连接就能成功响应。但如果调用了B机器的 `set` 接口，B机器上又没有这条连接，那么这条连接就无法响应。所以难点在于如何将同一个 `key` 的 `get` 、`set` 分配到一台机器。

做法有很多：
在做负载均衡的时候，就将连接分配到指定机器。刚开始我觉的很有道理，但细细想，如果这样做，在以后如果要加机器或减机器的时候会很麻烦。对横向的增减机器不友善。

最后我还是采用了leader给出的方案：用 `redis` 绑定 `key` 与机器的关系

即前端请求到一台机器上，以 `key` 做键，以机器 `IP` 做值放在 `redis` 里面。后端请求 `set` 接口时先用 `key` 去 `redis` 里面拿到机器 `IP` ，再将 `value` 发送到这台机器上。
此时就多了一个接口，用于机器内部相互调用


ChanSet
```go
func Router(){
    http.HandleFunc("/status/get", Get)
    http.HandleFunc("/status/set", Set)
    http.HandleFunc("/channel/set", ChanSet)
}

func ChanSet(w http.ResponseWriter, r *http.Request){
    ...
    key = ...
    value = ...
    Status[key] <- value
}
```

GET
```go
func Get(w http.ResponseWriter, r *http.Request){
    ...        
    IP = getLocalIp()       //得到本机IP
    RedisSet(key, IP)       //以key做键，IP做值放入redis
    Status[key] <- value
    ...
}
```
SET
```go
func Set(w http.ResponseWriter, r *http.Request){
    ...
    IP = RedisGet(key)    //用key去取对应机器的IP
    Post(IP, key, value) //将key与value都发送给这台机器
}
```

注这里相当于用 `redis sentinel` 做多台机器的通信。哨兵会帮我们将数据同步到所有机器上。 这样即可实现多机支持

## 跨域
刚部署到线上的时候，第一次尝试就跪了。查看错误...(`Access-Control-Allow-Origin`)...

因为前端是通过 `AJAX` 请求的长连接服务，所以存在跨域问题。

在服务端设置允许跨域

```go
func Get(w http.ResponseWriter, r *http.Request){
    ...
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Add("Access-Control-Allow-Headers", "Content-Type")
    ...
}
```

若是像微信的做法，动态的加载 `script` 方式，则没有跨域问题。

服务端直接允许跨域，可能会有安全问题，但我不是很了解，这里为了使用，就允许跨域了。

## Map并发读写问题

跨域问题解决之后，线上可以正常使用了。紧接着请测试同学压测了一下。

预期单机并发 `10000` 以上，测试同学直接压了 `10000` ，服务挂了。

可能预期有点高， `5000` 吧，于是压了 `5000` ，服务挂了。

1000呢，服务挂了。

100，服务挂了。

……

这下豁然开朗，不可能是机器问题，绝对是有BUG, 看了下报错

![map 并发读写](https://segmentfault.com/img/bVbm7wu?w=934&h=478)

去看了下[官方文档- Why are map operations not defined to be atomic?](https://golang.org/doc/faq#atomic_maps), `Map` 是不能并发的写操作，但可以并发的读。

原来对 `Map` 操作是这样写的

```go
func Get(w http.ResponseWriter, r *http.Request){
    ...
    Status[key] = make(chan string)
    defer close(Status[key])
    ...
    select {
    case <-ctx.Done():
        utils.ResponseJson(w, -1, "timeout", nil)
    case result := <-Status[key]:
        utils.ResponseJson(w, 0, "success", result)
    }
    ...
}

func ChanSet(w http.ResponseWriter, r *http.Request){
    ...
    Status[key] <- value
    ...
}
```

`Status[key] = make(chan string)` 在 `Status(map)` 里面初始化一个通道，是 `map` 的写操作

`result := <-Status[key]` 从 `Status[key]` 通道中读取一个值，由于是通道，这个值取出来后，通道内就没有了，所以这一步也是对 `map` 的写操作

`Status[key] <- value `向 `Status[key]` 内放入一个值，map的写操作

由于这三处操作的是一个map，所以要加同一把锁

```go
var Mutex sync.Mutex
func Get(w http.ResponseWriter, r *http.Request){
    ...
    //这里是同组大佬教我的写法，通道之间的拷贝传递的是指针，即statusChan与Status[key]指向的是同一个通道
    statusChan := make(chan string)
    Mutex.Lock()
    Status[key] = statusChan
    Mutex.Unlock()
    
    //在连接结束后将这些资源都释放
    defer func(){
        Mutex.Lock()
        delete(Status, key)
        Mutex.Unlock()
        close(statusChan)
        RedisDel(key)
    }()
    
    select {
        case <-ctx.Done():
            utils.ResponseJson(w, -1, "timeout", nil)
        case result := <-statusChan:
            utils.ResponseJson(w, 0, "success", result)
    }
    ...
}

func ChanSet(w http.ResponseWriter, r *http.Request){
    ...
    Mutex.Lock()
    Status[key] <- value
    Mutex.Unlock()
    ...
}
```

到现在，服务就可以正常使用了，并且支持上万并发。

## Redis过期时间

服务正常使用之后，leader review代码，提出redis的数据为什么不设置过期时间，反而要自己手动删除。我一想，对啊。

于是设置了过期时间并且将RedisDel(key)删了。设置完之后不出意外的服务跪了。

**究其原因**

我用一个 `key=1` 请求 `get` ，会在redis内存储一条数据记录(`1 => Ip`).如果我 `set` 了这条连接，按之前的逻辑会将 redis 里的这条数据删掉，而现在是等待它过期。若是在过期时间内，再次以这个 `key=1` ，调用 `set` 接口。 `set` 接口依然会从redis中拿到 `IP` ， `Post` 数据到 `ChanSet` 接口。而 `ChanSet` 中 `Status[key] <- value` 由于 `Status[key]` 是关闭的,会阻塞在这里，阻塞不要紧，但之前这里加了锁，导致整个程序都阻塞在这里。

这里和leader讨论过，仍使用redis过期时间但需要修复这个Bug

```go
func ChanSet(w http.ResponseWriter, r *http.Request){
    Mutex.Lock()
    ch := Status[key]
    Mutex.Unlock()

    if ch != nil {
        ch <- value
    }
}
```

不过这样有一个问题，就是同一个 `key` ，在过期时间内是无法多次使用的。不过这与业务要求并不冲突。

## Linux文件最大句柄数

在给测试同学测试之前，自己也压测了一下。不过刚上来就疯狂报错，“`%¥#@¥……%……%%..too many fail open...`”

搜索结果是linux默认最大句柄数 `1024` .

开了下自己的机器 `ulimit -a` 果然 `1024` 。修改（修改方法不多BB）

## 同时监听两个端口

服务有两个API， `get` 是给前端使用的，对外开放。 `set` 是给后端使用的，内部接口。所以这两个接口需要放在两个端口上。
由于 `http.ListenAndServe()` 本身有阻塞，故第一个监听需要一个 `goroutine` 

```go
go http.ListenAndServe(":11000", FrontendMux)    //对外开放的端口
http.ListenAndServe(":11001", BackendMux)    //内部使用的端口
```

# References

- [Golang-长连接-状态推送 - HammerMax](https://segmentfault.com/a/1190000017866100)

- [Why are map operations not defined to be atomic? - golang.org](https://golang.org/doc/faq#atomic_maps)

- [关于golang的context.WithTimeout的cancel的说明](https://blog.csdn.net/liangzhiyang/article/details/65633960)
