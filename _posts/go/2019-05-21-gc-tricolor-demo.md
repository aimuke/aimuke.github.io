---
title: "gc 三色标记法示例"
tags: [go, gc, "tricolor mark-and-sweep algorithm"]
list_number: n
---

**阅读本文前，建议先阅读[三色标记法原理介绍](http://localhost:4001/go/2019/05/21/gc-tricolor-algorithm/)**

Go的GC是如何实现并行的呢？其中的关键在于tricolor mark-and-sweep algorithm 三色标记清除算法。该算法能够让系统的gc暂停时间成为能够预测的问题。调度器能够在很短的时间内实现GC调度，并且对源程序的影响极小。下面我们看看三色标记清除算法是如何工作的：

假设我们有这样的一段链表操作的代码：

```go
var A LinkedListNode;
var B LinkedListNode;
// ...
B.next = &LinkedListNode{next: nil};
// ...
A.next = &LinkedListNode{next: nil};
*(B.next).next = &LinkedListNode{next: nil};
B.next = *(B.next).next;
B.next = nil;
```


# 1. 初始化
```go
var A LinkedListNode;

var B LinkedListNode;

// ...

B.next = &LinkedListNode{next: nil}
```
刚开始我们假设有三个节点A、B和C，作为根节点，红色的节点A和B始终都能够被访问到，然后进行一次赋值B.next = &C。初始的时候垃圾收集器有三个集合，分别为黑色，灰色和白色。现在，因为垃圾收集器还没有运行起来，所以三个节点都在白色集合中。

![gc 001](/assets/images/2019/0521/gc-demo-1.jpeg)

# 2. 新建对象
我们新建一个节点D,并将其赋值给A.next。即：
```go
var A LinkedListNode;

var B LinkedListNode;

// ...

B.next = &LinkedListNode{next: nil};
// ...
A.next = &LinkedListNode{next: nil};
```
需要注意的是，作为一个新的内存对象，需要将其放置在灰色区域中。为什么要将其放在灰色区域中呢？这里有一个规则，如果一个指针域发生了变化，则被指向的对象需要变色。因为所有的新建内存对象都需要将其地址赋值给一个引用，所以他们将会立即变为灰色。（这就需要问了，为什么C不是灰色？）

![gc2](/assets/images/2019/0521/gc-demo-2.jpeg)

# 3. 开始GC
在开始GC的时候，根节点将会被移入灰色区域。此时A、B、D三个节点都在灰色区域中。由于所有的程序子过程(process，因为不能说是进程，应该算是线程，但是在go中又不完全是线程)要么是程序正常逻辑，要么是GC的过程，而且GC和程序逻辑是并行的，所以程序逻辑和GC过程应该是交替占用CPU资源的。

![golang-gc3.001.jpeg](/assets/images/2019/0521/gc-demo-3.jpeg)

# 4. 遍历灰色对象
在扫描内存对象的时候，GC收集器将会把该内存对象标记为黑色，然后将其子内存对象标记为灰色。在任一阶段，我们都能够计算当前GC收集器需要进行的移动步数：`2*|white| + |grey|`,在每一次扫描GC收集器都至少进行一次移动，直到达到当前灰色区域内存对象数目为`0`。

![golang-gc4.001.jpeg](/assets/images/2019/0521/gc-demo-4.jpeg)

# 5. 新建对象
程序此时的逻辑为，新赋值一个内存对象E给C.next，代码如下：
```go
var A LinkedListNode;

var B LinkedListNode;

// ...

B.next = &LinkedListNode{next: nil};
// ...
A.next = &LinkedListNode{next: nil};
//新赋值 C.next = &E
*(B.next).next = &LinkedListNode{next: nil};
```
按照我们之前的规则，新建的内存对象需要放置在灰色区域，如图所示：

![golang-gc5.001.jpeg](/assets/images/2019/0521/gc-demo-5.jpeg)

这样做，收集器需要做更多的事情，但是这样做当在新建很多内存对象的时候，可以将最终的清除操作延迟。值得一提的是，这样处理白色区域的体积将会减小，直到收集器真正清理堆空间时再重新填入移入新的内存对象。

# 6. 指针重新赋值
程序逻辑此时将 B.next.next赋值给了B.next，也就是将E赋值给了B.next。代码如下：
```go
var A LinkedListNode;
var B LinkedListNode;
// ...
B.next = &LinkedListNode{next: nil};
// ...
A.next = &LinkedListNode{next: nil};
*(B.next).next = &LinkedListNode{next: nil};
// 指针重新赋值:
B.next = *(B.next).next;
```
这样做之后，如图所示，C将不可达。

![golang-gc6.001.jpeg](/assets/images/2019/0521/gc-demo-6.jpeg)

这就意味着，收集器需要将C从白色区域移除，然后在GC循环中将其占用的内存空间回收。

# 7. 不引用其他节点的灰色对象
将灰色区域中没有引用依赖的内存对象移动到黑色区域中，此时D在灰色区域中没有其它依赖，并依赖于它的内存对象A已经在黑色区域了，将其移动到黑色区域中。

![golang-gc7.001.jpeg](/assets/images/2019/0521/gc-demo-7.jpeg)

# 8. 删除节点引用
在程序逻辑中，将`B.next`赋值为了`nil`,此时E将变为不可达。但此时`E`在灰色区域，将不会被回收，那么这样**会导致内存泄漏吗？**其实不会，`E`将在下一个GC循环中被回收，三色算法能够保证这点：如果一个内存对象在一次GC循环开始的时候无法被访问，则将会被冻结，并在GC的最后将其回收。

![golang-gc8.001.jpeg](/assets/images/2019/0521/gc-demo-8.jpeg)

# 9. 对删除节点的处理
在进行第二次GC循环的时候，将E移入到黑色区域，但是C并不会移动，因为是C引用了E，而不是E引用C。

![golang-gc9.001.jpeg](/assets/images/2019/0521/gc-demo-9.jpeg)

# 10. 灰色对象处理
收集器再扫描最后一个灰色区域中的内存对象B，并将其移动到黑色区域中。

![golang-gc10](/assets/images/2019/0521/gc-demo-10.jpeg)

# 11. 回收白色区域
现在灰色区域已经没有内存对象了，这个时候就讲白色区域中的内存对象回收。在这个阶段，收集器已经知道白色区域的内存对象已经没有任何引用且不可访问了，就将其当做垃圾进行回收。而在这个阶段，E不会被回收，因为这个循环中，E才刚刚变为不可达，它将在下个循环中被回收。

![golang-gc11.001.jpeg](/assets/images/2019/0521/gc-demo-11.jpeg)

# 12. 区域变色
这一步是最有趣的，在进行下次GC循环的时候，完全不需要将所有的内存对象移动回白色区域，只需要将黑色区域和白色区域的颜色换一下就好了，简单而且高效。

![golang-gc12.001.jpeg](/assets/images/2019/0521/gc-demo-12.jpeg)

# GC三色算法小结

上面就是三色标记清除算法的一些细节，在当前算法下仍旧有两个阶段需要 `stop-the-world`：一是进行`root`内存对象的栈扫描；二是标记阶段的终止暂停。令人激动的是，标记阶段的终止暂停将被去除。在实践中我们发现，用这种算法实现的GC暂停时间能够在超大堆空间回收的情况下达到`<1ms`的表现。


# 参考文献

- [原文地址](https://segmentfault.com/a/1190000010753702#articleHeader14)

