---
title: "图解Golang的GC算法-三色标记法"
tags: [go, gc, "tricolor algorithm"]
---

**2019-03-04 [原文](https://i6448038.github.io/2019/03/04/golang-garbage-collector/)** 

本文结束后可以看一下[三色标记法示例](https://aimuke.github.io/go/2019/05/21/gc-tricolor-demo/)

虽然Golang的GC自打一开始，就被人所诟病，但是经过这么多年的发展，Golang的GC已经改善了非常多，变得非常优秀了。

以下是Golang GC算法的里程碑：

- v1.1 STW
- v1.3 Mark STW, Sweep 并行
- v1.5 三色标记法(tricolor algorithm)
- v1.8 hybrid write barrier

经典的GC算法有三种：引用计数(reference counting)、标记-清扫(mark & sweep)、复制收集(Copy and Collection)。

Golang的GC算法主要是基于标记-清扫(`mark and sweep`)算法，并在此基础上做了改进。因此，在此主要介绍一下标记-清扫(`mark and sweep`)算法，关于引用计数(reference counting)和复制收集(copy and collection)可自行百度。

# 标记-清扫(Mark And Sweep)算法
此算法主要有两个主要的步骤：

1. 标记(Mark phase): 找出不可达的对象，然后做上标记
2. 清除(Sweep phase): 回收标记好的对象

操作非常简单，但是有一点需要额外注意：`mark and sweep`算法在执行的时候，需要程序暂停！即`stop the world`。
也就是说，这段时间程序会卡在哪儿。故中文翻译成`卡顿`。

我们来看一下图解：

开始标记，程序暂停。程序和对象的此时关系是这样的：

![before-mark](/assets/images/2019/0521/mark_sweep_1.png)

然后开始标记，process找出它所有可达的对象，并做上标记。如下图所示：

![marked](/assets/images/2019/0521/mark_sweep_2.png)

标记完了之后，然后开始清除未标记的对象：

![sweep](/assets/images/2019/0521/mark_sweep_3.png)

然后垃圾清除了，变成了下图这样。

![after-sweep](/assets/images/2019/0521/mark_sweep_4.png)

最后，停止暂停，让程序继续跑。然后循环重复这个过程，直到process生命周期结束。

# 标记-清扫(Mark And Sweep)算法存在什么问题？
标记-清扫(`Mark And Sweep`)算法这种算法虽然非常的简单，但是还存在一些问题：
- `STW，stop the world`；让程序暂停，程序出现卡顿。
- 标记需要扫描整个heap
- 清除数据会产生heap碎片

这里面最重要的问题就是：mark-and-sweep 算法会暂停整个程序(`stw`)。

Go是如何面对并这个问题的呢？

# 三色并发标记法
我们先来看看Golang的三色标记法的大体流程。

首先：程序创建的对象都标记为白色。

![all-white](/assets/images/2019/0521/mark_sweep_5.png)

gc开始：扫描所有可到达的对象，标记为灰色

![mark-gray](/assets/images/2019/0521/mark_sweep_6.png)

从灰色对象中找到其引用对象标记为灰色，把灰色对象本身标记为黑色

![mark-black](/assets/images/2019/0521/mark_sweep_7.png)

监视对象中的内存修改，并持续上一步的操作，直到灰色标记的对象不存在

![all-marked](/assets/images/2019/0521/mark_sweep_8.png)

此时，gc回收白色对象。

![remove-black](/assets/images/2019/0521/mark_sweep_9.png)

最后，将所有黑色对象变为白色，并重复以上所有过程。

![set-white-again](/assets/images/2019/0521/mark_sweep_10.png)

好了，大体的流程就是这样的，让我们回到刚才的问题：Go是如何解决标记-清除(mark and sweep)算法中的卡顿(stw，stop the world)问题的呢？

# gc和用户逻辑如何并行操作？
标记-清除(`mark and sweep`)算法的`STW(stop the world)`操作，就是runtime把所有的线程全部冻结掉，所有的线程全部冻结意味着用户逻辑是暂停的。这样所有的对象都不会被修改了，这时候去扫描是绝对安全的。

Go如何减短这个过程呢？标记-清除(`mark and sweep`)算法包含两部分逻辑：标记和清除。

我们知道Golang三色标记法中最后只剩下的黑白两种对象，黑色对象是程序恢复后接着使用的对象，如果不碰触黑色对象，只清除白色的对象，肯定不会影响程序逻辑。所以：**清除操作和用户逻辑可以并发**。

标记操作和用户逻辑也是并发的，用户逻辑会时常生成对象或者改变对象的引用，那么标记和用户逻辑如何并发呢？

# process新生成对象的时候，GC该如何操作呢？不会乱吗？
我们看如下图，在此状态下：process程序又新生成了一个对象，我们设想会变成这样：

![new-object](/assets/images/2019/0521/mark_sweep_11.png)

但是这样显然是不对的，因为按照三色标记法的步骤，这样新生成的对象A最后会被清除掉，这样会影响程序逻辑。

Golang为了解决这个问题，引入了`写屏障 write barrier`这个机制。

**写屏障write barrier**：该屏障之前的写操作和之后的写操作相比，先被系统其它组件感知。

通俗的讲：就是在`gc`跑的过程中，可以监控对象的内存修改，并对对象进行重新标记。(实际上也是超短暂的`stw`，然后对对象进行标记)

在上述情况中，**新生成的对象，一律都标位灰色！**
即下图：

![set-new-to-gray](/assets/images/2019/0521/mark_sweep_12.png)

# 灰色或者黑色对象的引用改为白色对象的时候，Golang是该如何操作的？
看如下图，一个黑色对象引用了曾经标记的白色对象。

![reference-change](/assets/images/2019/0521/mark_sweep_13.png)

这时候，写屏障机制被触发，向GC发送信号，GC重新扫描对象并标位灰色。

![mark-changed-reference](/assets/images/2019/0521/mark_sweep_14.png)

因此，gc一旦开始，无论是创建对象还是对象的引用改变，都会先变为灰色。

# 参考文献

- [原文地址](https://i6448038.github.io/2019/03/04/golang-garbage-collector/)

- [Golang’s Real-time GC in Theory and Practice](https://making.pusher.com/golangs-real-time-gc-in-theory-and-practice/)

- [Golang’s realtime garbage collector](https://www.youtube.com/watch?v=n59VtiRx34s&t=1151)

- [Golang 垃圾回收剖析](https://mp.weixin.qq.com/s/Wh7A6czDIRofxveY_bEDNQ)

