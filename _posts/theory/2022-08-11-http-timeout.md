---
title: "http 请求 timeout 分析"
tags: [http, timeout, tcp, connection, socket]
list_number: n
---

# Http 请求request分析

如果你的 Nginx 服务器流量足够大，足够繁忙。可能你会在 Nginx 的 error log 中看到下面这样的日志：

![Alt text](https://ms2008.github.io/img/in-post/nginx\_timeout.png)

如果你去仔细观察，会发现这两个的 timeout 似乎有些不太一样：

> * 110: Connection timed out
> * timeout

那这两种 timeout 有什么区别？分别在什么情况下会发生？

首先无论是哪种语言，不管是客户端还是服务端，在 TCP 编程中通常都可以为 sock 设置一个 timeout 时间。而这个 timeout 又可以细分为 connect timeout、read timeout、write timeout。read timeout 和 write timeout 必须是在 connect 之后才能发生，今天不做过多讨论。上面那两种 timeout 均属于 connect timeout。

另外我们需要补充下 TCP 重传机制的相关知识：

我们知道在 TCP 的三次握手中，Client 发送 SYN，Server 收到之后回 SYN\_ACK，接着 Client 再回 ACK，这时 Client 便完成了 connect() 调用，进入 ESTAB 状态。如果 Client 发送 SYN 之后，由于网络原因或者其他问题没有收到 Server 的 SYN\_ACK，那么这时 Client 便会重传 SYN。重传的次数由内核参数 `net.ipv4.tcp_syn_retries` 控制，重传的间隔为 \[1,3,7,15,31]s 等。(有很多文章都说是 \[3,6,9]s 等，不过根据我的抓包分析，貌似不太符合，如下图)

![Alt text](https://ms2008.github.io/img/in-post/syn\_retry.png)

**如果 Client 重传完所有 SYN 之后依然没有收到 SYN\_ACK，那么这时 connect() 调用便会抛出 connection timeout 错误。如果 Client 在重传 SYN 期间，Client 的 sock timeout 时间到了，那么这时 connect() 会抛出 timeout 错误。**

****

# **Reference**

* [原文 理解 timeout，这一篇就够了](https://ms2008.github.io/2017/04/14/tcp-timeout/)
