---
title: "redis的多库"
tags: [redis, database]
---

# 不要将 redis 和 mysql 混为一谈
在接触 redis 之前，相信很多人都有 mysql 的使用经验。

mysql 的实体分层由上至下依次是：

- 实例（instance）mysqld 进程。
- 库（database）
- 表（table）
- 记录（row）
- 字段（field）

redis 的实体分层由上至下依次是：

- 实例（instance）redis 进程。
- 库（database）
- 键值（Key-Value）

我们很容易将 mysql 与 redis 的实例（instance）和库（database）等同起来，然而却大错特错。

**其实**

- redis 的实例（instance） 等同于 mysql 的库（database）
- redis 的库（database） 等同于 mysql 的表（table）

# redis 的多库是鸡肋
redis 是无预定义结构的（schema-less）数据库，表（table ）的存在意义不大，它更多地是做为命名空间（name space），由于使用的是很不友好的数字命名（默认 0-15）， redis 中的 库（database）形同鸡肋。

以下为 redis 作者的观点，引用自 [database names? - Google 网上论坛](https://groups.google.com/forum/#!msg/redis-db/vS5wX8X4Cjg/8ounBXitG4sJ)

> I understand how this can be useful, but unfortunately I consider Redis multiple database errors my worst decision in Redis design at all… without any kind of real gain, it makes the internals a lot more complex. The reality is that databases don't scale well for a number of reason, like active expire of keys and VM. If the DB selection can be performed with a string I can see this feature being used as a scalable O(1) dictionary layer, that instead it is not.
>
>With DB numbers, with a default of a few DBs, we are communication better what this feature is and how can be used I think. I hope that at some point we can drop the multiple DBs support at all, but I think it is probably too late as there is a number of people relying on this feature for their work.

# 使用 redis 多库是妥协的结果
有一种观点认为不同的应用（app）使用不同的库（database）可以从而避免键命名冲突。

redis 对于不同的库（database）没有提供任何隔离机制，完全依赖于应用（app）部署时约定使用不同的库（database）。

为什么不约定应用（app）使用的所有键都加上应用（app）前缀，或者每个应用（app）使用不同的 redis 实例（instance）呢？

**从开发人员的角度来说**

- 如果每个应用（app）使用不同的实例（instance），是最省事的，连接 redis 后，直接操作即可
- 如果每个应用（app）使用不同的库（database），略微麻烦一点，连接 redis 后，先执行一下 select db ，redis 客户端库会提供支持
- 如果每个应用（app）的键（key）都加上应用（app）前缀，会很麻烦，每一处访问 redis 的代码都要涉及

**从运维人员的角度来说**

- 如果每个应用（app）使用不同的实例（instance），是最麻烦的，维护压力剧增，每个实例（instance）背后还要有配套的启动、停止脚本，监控，主备实例等
- 如果每个应用（app）使用不同的库（database），略微麻烦一点，分配并记录一下，告知开发人员使用指定的 redis 实例及库（database）
- 如果每个应用（app）的键（key）都加上应用（app）前缀，是最省事的，可以灵活地为应用（app）安排使用实例（instance）或库（database）

**不同的应用（app）使用不同的库（database）这一方案被采用，很可能是开发人员与运维人员互相妥协的结果。**

# redis 的多库扩容难
redis 的数据量或者请求数过高，会导致 redis 不稳定，最终影响服务质量。

这时候就要考虑 redis 扩容了，需要将其中一个库（database）迁到新的实例（instance）上，过程如下：

1. 停掉应用（app）
2. 将应用（app）的 redis 库（database）同步到新的 redis 实例（instance）上

   通过拷贝 dump.rdb 的方式同步（传输）数据，redis 实例（instance）上的所有库（database）数据都是混在一起的，其它应用（app）数据会增加数据同步（传输）时间及新 redis 实例（instance） 数据载入时间。

   如果通过工具在线将应用（app）的 redis 库（database）拷贝到运行中的新 redis 实例（instance）上，会很耗时，对本身负载就很高的 redis 添加更多压力，可能会影响其它应用（app）。

   新 redis 实例（instance）设置为旧 redis 实例（instance） 的 slave，同步完成后取消 Master-Slave 关系，这种方法相对更好一些。

3. 启动新的 redis 实例
4. 修改应用（app）配置指向新的 redis 实例
5. 启动应用（app）
6. 清除两个 redis 实例中的脏数据

   select db ，然后执行 flushdb 命令即可，这可能是使用多库最大的好处。

应用（app）的停机时间（Down Time）肯定短不了。

如果每个应用（app）使用不同的实例，需要将某个实例迁到新机器，则可以做到平滑扩容（迁移），过程如下：

1. 应用（app）使用 redis sentinel 方式访问 redis
2. 新机器部署 redis 新实例
3. 新实例设置为旧 redis 实例的 slave
4. 同步完成后进行 Master-Slave 换位
5. 应用（app）会自动切到新的 redis 实例
6. 旧的 redis 实例可以停掉


# 多库的特点
注意：Redis支持多个数据库，并且每个数据库的数据是隔离的不能共享，并且基于单机才有，如果是集群就没有数据库的概念。

Redis是一个字典结构的存储服务器，而实际上一个Redis实例提供了多个用来存储数据的字典，客户端可以指定将数据存储在哪个字典中。这与我们熟知的在一个关系数据库实例中可以创建多个数据库类似，所以可以将其中的每个字典都理解成一个独立的数据库。

每个数据库对外都是一个从0开始的递增数字命名，Redis默认支持16个数据库（可以通过配置文件支持更多，无上限），可以通过配置`databases`来修改这一数字。客户端与Redis建立连接后会自动选择`0`号数据库，不过可以随时使用`SELECT`命令更换数据库，如要选择`1`号数据库：

```sh
redis> SELECT 1
OK
redis [1] > GET foo
(nil)
```
然而这些以数字命名的数据库又与我们理解的数据库有所区别。

- 首先Redis不支持自定义数据库的名字，每个数据库都以编号命名，开发者必须自己记录哪些数据库存储了哪些数据。
- 另外Redis也不支持为每个数据库设置不同的访问密码，所以一个客户端要么可以访问全部数据库，要么连一个数据库也没有权限访问。
- 最重要的一点是多个数据库之间并不是完全隔离的，比如FLUSHALL命令可以清空一个Redis实例中所有数据库中的数据。

综上所述，`这些数据库更像是一种命名空间，而不适宜存储不同应用程序的数据`。比如可以使用0号数据库存储某个应用生产环境中的数据，使用1号数据库存储测试环境中的数据，但不适宜使用0号数据库存储A应用的数据而使用1号数据库B应用的数据，不同的应用应该使用不同的Redis实例存储数据。**由于Redis非常轻量级，一个空Redis实例占用的内在只有1M左右，所以不用担心多个Redis实例会额外占用很多内存**。

# 参考文献

- [原文](http://blog.kankanan.com/article/52ff7528-redis-7684591a5e93.html)

- [多库的特点](http://www.runoob.com/note/35579)

- [What's the Point of Multiple Redis Databases? - Stack Overflow](http://stackoverflow.com/questions/16221563/whats-the-point-of-multiple-redis-databases)

- [database names? - Google 网上论坛](https://groups.google.com/d/msg/redis-db/vS5wX8X4Cjg/8ounBXitG4sJ)

- [Partitioning: how to split data among multiple Redis instances. – Redis](http://redis.io/topics/partitioning)

- [Redis Presharding](http://oldblog.antirez.com/post/redis-presharding.html)
