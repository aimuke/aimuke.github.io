---
title: "raft一致性算法详解"
tags: [algorithm, raft, todo]
--- 

在现实的分布式系统中，不能可能保证集群中的每一台机器都是100%可用可靠的，集群中的任何机器都可能发生宕机、网络连接等问题导致集群中的某个节点不可用，这样，那个节点的数据就有可能和集群不一致，所以需要有一种机制，来保证在大多数机器都存在的情况下向外提供可靠的数据服务。这里的大多数节点指的是 `集群半数以上` 的节点。

`raft算法` 就是一种在分布式系统中解决集群中多节点之间数据一致性的算法。Golang生态圈中大名鼎鼎的 `etcd` 就是使用的raft算法来保持数据一致性的，与raft类似的一致性算法还有 `Paxos算法`、`Zab协议`等。

其实，raft算法维持数据一致性的核心思想很简单，就是：`少数服从多数`。

# leader选举

保证数据一致性最好的方式就是只有唯一的一个节点，唯一的这个节点读，唯一的这个节点写，这样数据肯定是一致的；但是分布式架构显然不可以一个节点，于是，raft算法提出，在集群的所有节点中，需要有一个节点来充当这一个唯一的节点，在一段时间内，只有这一个节点负责读写数据，然后其他节点同步数据。这个唯一的节点叫 `leader` 节点，其他负责同步数据的节点叫做 `follower` 节点。在集群中，还会有其他状态的节点，例如 `candidate` 节点，这种节点只有在选举 `leader` 的时候才会有。

节点的 `leader` 选举和现实生活中的选举十分类似，就是投票，集群中获票数最多的那个，就是 `leader` 节点，所以为防止出现平局的情况(平局的情况也有解决方案，下文会说)，一般在部署节点的时候，会将节点数设置为奇数(`2n+1`)。

## 三种状态
这些节点是如何选举的呢？我们先从 `follower` 、 `leader` 、 `candidate` 这三种状态说起。

在集群中，有三个节点 A 、B、C。在集群刚刚开始的时候，他们仨都是 `follower` 。
```
                   A (follower)


        B(follower)              C(follower)
```

过一段时间后，A变成了 `candidate` ，这是要选举了！
```
                   A (follower -> candidate)


        B(follower)              C(follower)
```

## 超时时间

为啥A能变成 `candidate` ？凭啥？因为A的 `election timeout` 到期了，`election timeout` 是选举超时时间，集群中的每个节点都有一个`election timeout`，每个节点的`election timeout`都是150ms ~ 300ms之间的一个随机数。每当节点的`election timeout`时间到了，就会触发节点变为 `candidate` 状态。A的选举超时时间到了，所以A理所当然变为了 `candidate` 。

所以，我们知道，其实A、B、C三个节点除了有状态，还有个选举超时时间 `election timeout`

```
                     A (candidate) 
                     election timer = 0
                    

   B(follower)                       C(follower )
   election timer= 200ms             election timer= 50ms
```


此时， `candidate` 节点A会向整个集群发起选举投票，它会先投自己一票，然后告诉B、C 大选开始了！

```
                     A (candidate) 
                     election timer = 0
                     得票数 = 1
                    /              \
                   / request vote   \ 
                  ↙                  ↘
   B(follower)                       C(follower )
   election timer= 200ms             election timer= 50ms
```

**注意！只有 `candidate` 状态的节点，才可以参加竞选变为 `leader` ，B、C这两个 `follower` 是没有资格的！**

## term 任期
除此之外，每个节点中还有一个字段，叫 `term` 意思就是任期，和美国大选的第几期总统差不多一个意思，这个 `term` 是一个全局的、连续递增的整数，每进行一次选举， `term` 就会加一，如果 `candidate` 赢得选举，它会当 `leader` 直到此次任期结束。

此时，A触发了选举，它的term应该是加一的。

```
                     A (candidate) 
                     election timer = 0
                     得票数 = 1, term = term + 1
                    /              \
                   / request vote   \ 
                  ↙                  ↘
   B(follower)                       C(follower )
   election timer= 200ms             election timer= 50ms
```

当B、C收到A发出的大选消息后，B、C开始投票，此时只有A这一个 `candidate` ，所以理所当然发消息都只能投A。

```
                     A (candidate) 
                     election timer = 0
                     得票数 = 3, term = term + 1
                    ↗              ↖
                   / request vote   \ 
                  /                  \
   B(follower)                       C(follower )
   election timer= 200ms             election timer= 50ms
```

此时A当选leader!

为了巩固自己的“统治”，防止A在任期之间其他节点因为自身 `election timout` 而触发选举， `leader` 节点A会不定时的向两个 `follower` 节点B、C发送心跳消息，B和C收到心跳消息后，会重置 `election timout` 。心跳检测时间很短，要远远小于选举超时时间 `election timout`。


```
                      A (leader) 
                    /             \
                   /     心跳检测   \ 
                  ↙                  ↘
   B(follower)                       C(follower )
   election timer= 200ms             election timer= 50ms
```
B、C收到心跳检测后，返回心跳响应，并重置超时时间 `election timeout`。

假设A发送的心跳检测消息由于网络原因例如延迟、丢包等等没有传送到B、C中的某个 `follower` 节点，而此时这个节点刚好 `election timeout` ，则触发选举。
C修改自身节点任期值 `term` 为2，自身状态变为 `candidate` ，且投自身一票后，发起选举！


```
                      A (leader)   
                      term = 1
                                ↖
                                  \
                     request vote  \ 
                                    \
   B(follower)        <------------- C(candidate )
   election timer= 200ms             election timer= 0
   term = 1                          term = term + 1 = 1+1
```

这时候，由于C的任期值 `term` 变为2大于A的，在raft协议中，但收到任期值大于自身的节点，都会更改自身节点的 `term` 值，并切换为 `follower` 状态并重置`election time`。因此，这时候A由 `leader` 直接变为 `follower`！
```
                      A (follower)   
                      term = 2
                                ↖
                                  \
                     request vote  \ 
                                    \
   B(follower)        <------------- C(candidate )
   election timer= 200ms             election timer= 0
   term = 2                          term = term + 1 = 1+1
```

## 多candidate
我们再来考虑一种极端情况：**假设有偶数个节点，并且同时有两个节点进入 `candidate` 状态！**

例如有以下四个节点A、B、C、D。A和B同时进入了 `cadidate` 状态并开始选举。

```
      A(candidate)               B(candidate)


      C(follower)                D(follower)
```

假如A和B中任意一个获得了超过半数以上的多数票，则变为 `leader` ！


```
      A(leader)               B(follower)


      C(follower)                D(follower)
```

但是假如两个经过一次选举后得的票数相同或者都没有超过半数，则宣告选举失败并结束！等待A和C这两个 `candidate` 节点中任意一个节点的 `election time` 超时，然后发起新一轮选举。

注意：**虽然票数相同或者都没有超过半数导致的选举失败了，但是任期值term还是要叠加的！**

A、B票数相同，等待哪个先超时。

```
      A(candidate)               B(candidate)
      term = 4                    term = 4


      C(follower)                D(follower)
      term = 4                    term = 4
      vote for A                  vote for B
```

此时A先超时。则A发起选举，由于A `term` 值显然是最大的，则A会最终当选为`leader`。

```
      A(candidate)   -------->    B(candidate)
      term = 5                    term = 4
         |               \
         | request vote    \
         |                   \
         ↓                     ↘
      C(follower)                D(follower)
      term = 4                    term = 4
      vote for A                  vote for B
```

# 日志复制

当`leader`选出来后，无论读和写都会由`leader`节点来处理。

是的，读也由leader来处理，leader拿到请求后，再决定由哪一个节点来处理，要么将请求分发，要么自己处理；即使client端请求的是 `follower` 节点， `follower` 节点也会现将请求信息转给 `leader` ，再由 `leader` 决定由哪个节点来处理。

## 同步过程
下面来说说写的情况：

以下有A、B、C三个节点，其中A是leader节点

当client请求过来要求写操作的时候，leader A先把数据写在本身节点的log文件中

```
                                   1.写本节点
  client    ------>  A(leader)   -------------> log        
                 /            \
               /               \
             / 2.append entries  \
           ↙                       ↘
  B(follower)                    C(follower)

```

然后A将发 `append entries` 消息发送给B、C节点。

注意！`append entries` 消息其实是根据节点的不同而消息也不同的，因为集群中数据可能不一致，一味的传相同数据，显然不可以。具体怎么不一致，稍后再说。

B、C再收到消息后，把数据添加到本地，然后向A发消息，告诉A已经收到。

```
                                  
                  A(leader)           
                 ↗        ↖ 
               /             \
             /     响应        \
           /                     \
  B(follower)                    C(follower)
  保存到本地                      保存到本地

```

`leader` A收到后，先提交记录，然后返回客户端响应。`leader` 确保至少集群中超过半数节点已接收到数据后再向 Client 确认数据已接收。一旦向 Client 发出数据接收 Ack 响应后，表明此时数据状态进入已提交（Committed），Leader 节点再向 Follower 节点发通知告知该数据状态已提交。
```
            2.响应                       1.commit
  client    <------  A(leader)   -------------> log        
                /            \
               /               \
             / 3.commit          \
           ↙                       ↘
  B(follower)                    C(follower)

```

然后，leaderA继续向B、C两个follower发送写数据commit的通知。

B、C两个节点收到通知后，先commit自身的log数据，然后再通知leaderA已更新结束。

```
                                  
                  A(leader)           
                 ↗        ↖ 
               /             \
             /     2.响应      \
           /                     \
  B(follower)                    C(follower)
  1.commit                      1.commit

```

到此整个数据同步也就结束了。

每次写数据，都需要先更新，然后commit。每个节点中都有两个索引，一个是当前提交的索引值 `commitIndex` ，一个是目前数据的最后一行索引值 `lastApplied` 。

```
           ------------
           |     1     |
           ------------
           |     2     |
           ------------
           |     3     |     <------- lastApplied
           ------------
           |     4     |     <------- commitIndex
           ------------
           |           |
           ------------
```

而leader节点中，除了需要存储自身节点的 `commitIndex` 和 `lastApplied` 之外，还需要知道所有follower的存储情况，因而leader节点中多了一张表，这张表中记录了所有follower节点的存储情况，这张表中有两个属性，一个属性叫 `nextIndex` 记录的是follower节点没有的数据索引，需要发送 `append entries` 的数据索引；还有一个 `matchIndex` 记录的是leader节点已知的follower节点的数据。如下图所示:

```
  A(leader)                   --------------------------------
  --------------             |                   ↓   ↧
| 1  | 2 | 3 |               |    A  | 1 | 2 | 3 |   
  --------------             |                   ↓   ↧
                             |    B  | 1 | 2 | 3 |
                             |           ↓           ↧
                             |    C  | 1 |
                             | 注: (↧: nextIndex, ↓: matchIndex)
                              ---------------------------------


  B(follower)                   C(follower)
  --------------               --------------
| 1  | 2 | 3 |                | 1 |
  --------------               --------------
```

因此，当数据更新的时候， `leader` A 向节点B、C发送不同的 `append entries`。

```
  A(leader)                   --------------------------------
  --------------             |                   ↓   ↧
| 1  | 2 | 3 |               |    A  | 1 | 2 | 3 | 4 | 
  --------------             |                   ↓   ↧
                             |    B  | 1 | 2 | 3 | 4 |
                             |           ↓           ↧
                             |    C  | 1 |         4 |
                             | 注: (↧: nextIndex, ↓: matchIndex)
                              ---------------------------------
     |             \
     |               \
     ↓                  ↘
   增加4                增加2,3,4
  B(follower)                   C(follower)
  --------------               --------------
| 1  | 2 | 3 |                | 1 |
  --------------               --------------
```

## leader 更新
当A节点不再当 `leader` 时，其他节点并不能知道 leader A保存的 `matchIndex` 和 `nextIndex` 这两个数组的数据。当其他节点成功当选为 `leader` 节点后，会将 `nextIndex` 全部重置为自身的 `commitIndex` ，而 `matchIndex` 则全部重置为`0`。如下图：

```
  B(leader)                   --------------------------------
  --------------             |       ↓               ↧
| 1  | 2 | 3 | 4 |           |    A  | 1 | 2 | 3 | 4 |  
  --------------             |       ↓               ↧
                             |    B  | 1 | 2 | 3 | 4 |
                             |       ↓               ↧
                             |    C  | 1 | 2 | 3 | 4 |
                             | 注: (↧: nextIndex, ↓: matchIndex)
                              ---------------------------------


  A(follower)                   C(follower)
  --------------               --------------
| 1  | 2 | 3 |                | 1 | 2 |
  --------------               --------------
```

则，leaderB会向A和C节点发append entries，去”补全”数据
```
  B(leader)                   --------------------------------
  --------------             |       ↓               ↧
| 1  | 2 | 3 | 4 |           |    A  | 1 | 2 | 3 | 4 |  
  --------------             |       ↓               ↧
                             |    B  | 1 | 2 | 3 | 4 |
                             |       ↓               ↧
                             |    C  | 1 | 2 | 3 | 4 |
                             | 注: (↧: nextIndex, ↓: matchIndex)
                              ---------------------------------
    |               \
     |                \
     ↓                  ↘
   增加1,2,3,4                增加1,2,3,4

  A(follower)                   C(follower)
  --------------               --------------
| 1  | 2 | 3 |                | 1 | 2 |
  --------------               --------------
```


节点收到请求后，如果存在数据，就不动直接返回，如果没有数据则缺哪个补哪个。

## 故障保障

Raft 协议强依赖 `leader` 节点的可用性来确保集群数据的一致性。数据的流向只能从 `Leader` 节点向 `Follower` 节点转移。当 `Client` 向集群 `Leader` `节点提交数据后，Leader` 节点接收到的数据处于未提交状态（`Uncommitted`），接着 `Leader` 节点会并发向所有 `Follower` 节点复制数据并等待接收响应，确保至少集群中超过半数节点已接收到数据后再向 `Client` 确认数据已接收。一旦向 `Client` 发出数据接收 `Ack` 响应后，表明此时数据状态进入已提交（`Committed``），Leader` 节点再向 `Follower` 节点发通知告知该数据状态已提交。

![数据同步流程](https://images2015.cnblogs.com/blog/815275/201603/815275-20160301175358173-526445555.png)

在这个过程中，主节点可能在任意阶段挂掉，看下 Raft 协议如何针对不同阶段保障数据一致性的。

1. 数据到达 `Leader` 节点前

   这个阶段 `Leader` 挂掉不影响一致性，不多说。

2. 数据到达 `Leader` 节点，但未复制到 `Follower` 节点

   这个阶段 `Leader` 挂掉，数据属于未提交状态， `Client` 不会收到 `Ack` 会认为超时失败可安全发起重试。 `Follower` 节点上没有该数据，重新选主后 `Client` 重试重新提交可成功。原来的 `Leader` 节点恢复后作为 `Follower` 加入集群重新从当前任期的新 `Leader` 处同步数据，强制保持和 `Leader` 数据一致。

3. 数据到达 `Leader` 节点，成功复制到 `Follower` 所有节点，但还未向 `Leader` 响应接收

   这个阶段 `Leader` 挂掉，虽然数据在 `Follower` 节点处于未提交状态（`Uncommitted`）但保持一致，重新选出 `Leader` 后可完成数据提交，此时 `Client` 由于不知到底提交成功没有，可重试提交。针对这种情况 Raft 要求 RPC 请求实现幂等性，也就是要实现内部去重机制。

4. 数据到达 `Leader` 节点，成功复制到 `Follower` 部分节点，但还未向 `Leader` 响应接收

   这个阶段 `Leader` 挂掉，数据在 Follower 节点处于未提交状态（`Uncommitted`）且不一致，Raft 协议要求投票只能投给拥有最新数据的节点。所以拥有最新数据的节点会被选为 Leader 再强制同步数据到 Follower，数据不会丢失并最终一致。

5. 数据到达 `Leader` 节点，成功复制到 `Follower` 所有或多数节点，数据在 `Leader` 处于已提交状态，但在 `Follower` 处于未提交状态

   这个阶段 Leader 挂掉，重新选出新 Leader 后的处理流程和阶段 3 一样。

6. 数据到达 `Leader` 节点，成功复制到 `Follower` 所有或多数节点，数据在所有节点都处于已提交状态，但还未响应 `Client`

   这个阶段 Leader 挂掉，Cluster 内部数据其实已经是一致的，Client 重复重试基于幂等策略对一致性无影响。

7. 网络分区导致的脑裂情况，出现双 `Leader`

   网络分区将原先的 `Leader` 节点和 Follower 节点分隔开， `Follower` 收不到 `Leader` 的心跳将发起选举产生新的 Leader。这时就产生了双 `Leader`，原先的 `Leader` 独自在一个区，向它提交数据不可能复制到多数节点所以永远提交不成功。向新的 `Leader` 提交数据可以提交成功，网络恢复后旧的 `Leader` 发现集群中有更新任期（`Term`）的新 `Leader` 则自动降级为 `Follower` 并从新 Leader 处同步数据达成集群数据一致。

综上穷举分析了最小集群（3 节点）面临的所有情况，可以看出 Raft 协议都能很好的应对一致性问题，并且很容易理解。

# 总结
触发选举的唯一条件是 `election timeout`，心跳超时等其他条件仅仅是触发了非 `leader` 节点的 `election timeout`。

节点选举的时候，`term` 值大的一定会力压 `term` 值小的当选 `leader`。

`leader` 节点向 `follower` 节点中发送 `append entries` 的时候，并不是缺少`1、2、3` 就直接发送 `1、2、3` 而是分批次，一次发送一条。`1！ 2！ 3！`三条数据，分三次发完。(怕图片误导，特此说明!)


# Reference

- [原文 - raft一致性算法详解](https://i6448038.github.io/2018/12/12/raft/)

- [Raft 为什么是更易理解的分布式一致性算法](https://www.cnblogs.com/mindwind/p/5231986.html)
