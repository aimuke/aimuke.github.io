---
title: "k8s client-go与controller介绍"
tags: [k8s, crd, controller, todo]
---


我们知道，Kubernetes 中一切都可视为资源，它提供了很多默认资源类型，如  `Pod、Deployment、Service、Volume` 等一系列资源，能够满足大多数日常系统部署和管理的需求。但是，在一些特殊的需求场景下，这些现有资源类型就满足不了，那么这些就可以抽象为 Kubernetes 的自定义资源，在 Kubernetes 1.7 之后增加了对 `CRD(CustomResourceDefinition)` 自定义资源二次开发能力来扩展 Kubernetes API，通过 `CRD` 我们可以向 Kubernetes API 中增加新资源类型，而不需要修改 Kubernetes 源码或创建自定义的 `API server`，该功能大大提高了 Kubernetes 的扩展能力。它是 `(TPR) ThirdPartyResource` 的替代者，在 1.9 以上版本 `TPR` 将被废弃。

下图展示了 `client-go` 库各组件如何工作以及同自定义 `controller` 的交互。 

![client-controller](http://www.iceyao.com.cn/img/posts/2019-01-15/1.png)

通过图示，我们可以看到几个核心组件以及交互流程，以上蓝色部分是 `client-go` 组件，黄色部分是自定义 `Controller` 组件，各组件作用介绍如下：

# client-go

**Reflector**：该组件是用来监测指定资源类型的 Kubernetes API，当监测 API 接收到新资源类型实例变化时，它将通过 List API 来获取新创建的 `Object` ，并将其放入到 `Delta Fifo queue`（一个先进先出队列）中。在 `ListAndWatch` 机制下，一旦 `APIServer` 端有新的实例被创建、删除或者更新， `Reflector` 都会收到“事件通知”。这时，该事件及它对应的 API 对象这个组合，就被称为`增量（Delta）`，它会被放进一个 `Delta FIFO Queue（即：增量先进先出队列）` 中。

**Informer**：该组件是用来将 `Delta Fifo queue` 中的 `Object` 循环取出，并且保存 `Object` 供后边索引，并调用我们自定义 `Controller` 传递 `Object` 。

**Indexer**：该组件是为 `Object` 提供索引功能，典型的用例就是通过 `Object` 的标签创建索引，并且使用线程安全的数据存储来存储 `Object` 以及它的 `Keys` 。

# Custom Controller

**Informer reference**：该组件是知道如何使用自定义资源 `Object` 的 `Informer` 实例的引用，我们需要在自定义 `Controller` 代码中创建适当的 `Informer` 。

**Indexer reference**：该组件是知道如何使用自定义资源 `Object` 的 `Indexer` 实例的引用，我们需要在自定义 `Controller` 代码中创建适当的 `Indexer` ，并且将使用该引用处理后续检索 `Object` 。

**Resource Event Handlers**：该组件是当 `Informer` 要部署 `Object` 到我们自定义 `Controller` 时，调用的 `Callback` 函数。这些函数可以获取被调度 `Object` 的 `Key` ，并将 `Key` 存入工作队列以便进行下一步处理。

**Work queue**：该组件是我们自定义 `Controller` 中创建用来解耦一个处理中的 `Object` ，也是上边 `Resource Event Handlers` 存储 `Key` 的地方。

**Process Item**：该组件是我们自定义 `Controller` 中创建用来处理 `Work queue` 的一些列函数，这些方法通常使用 `Indexer reference` 并检索该 `Object` 对应的 `Key` 。

# 小结
简单的说，整个处理流程大概为： `Reflector` 通过检测 Kubernetes API 来跟踪该扩展资源类型的变化，一旦发现有变化，就将该 `Object` 存储队列中， `Informer` 循环取出该 `Object` 并将其存入 `Indexer` 进行检索，同时触发 `Callback` 回调函数，并将变更的 `Object Key` 信息放入到工作队列中，此时自定义 `Controller` 里面的 `Process Item` 就会获取工作队列里面的 `Key` ，并从 `Indexer` 中获取 `Key` 对应的 `Object` ，从而进行相关的业务处理。

# Reference

- [原文- Kubernetes CRD (CustomResourceDefinition) 自定义资源类型](https://blog.csdn.net/aixiaoyang168/article/details/81875907)
