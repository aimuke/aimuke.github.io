---
title: "pilot如何与envoy交互"
tags: [istio, pilot, envoy, pilot-discovery]
---

pilot 中与 envoy 交互的使用的是 enovy 提供的 [The universal data plane API](https://blog.envoyproxy.io/the-universal-data-plane-api-d15cec7a). 其对应的代码在 `istio/pilot/pkg/proxy/envoy` 中。

# data-plane-api

`data-plane-api` v1 版本的时候使用的 `json/rest` 接口，使用轮询的方式来实现数据的同步。在 v2 版本的时候与 google 合作使用了 `grpc` 接口的方式来进行定义，使用双向流的方式极大的提升了性能。 v2 版本是 v1 版本的演进，其兼容了 v1 版本的 `rest` 接口。

v2 API 由以下部分组成：

- `Endpoint Discovery Service (EDS)`：这是v1 SDS API的替代品。SDS是一个不幸的名字选择，所以我们正在v2中修复这个问题。此外，gRPC的双向流性质将允许将负载/健康信息报告回管理服务器，为将来的全局负载均衡功能开启大门。
- `Cluster Discovery Service (CDS)`：和v1没有实质性变化。
- `Route Discovery Service (RDS)`：和v1没有实质性变化。
- `Listener Discovery Service (LDS)`：和v1的唯一主要变化是：我们现在允许监听器定义多个并发过滤栈，这些过滤栈可以基于一组监听器路由规则（例如，SNI，源/目的地IP匹配等）来选择。这是处理“原始目的地”策略路由的更简洁的方式，这种路由是透明数据平面解决方案（如Istio）所需要的。
- `Health Discovery Service (HDS)`：该 API 将允许 Envoy 成为分布式健康检查网络的成员。中央健康检查服务可以使用一组 Envoy 作为健康检查终点并将状态报告回来，从而缓解N²健康检查问题，这个问题指的是其间的每个 Envoy 都可能需要对每个其他 Envoy 进行健康检查。
- `Aggregated Discovery Service (ADS)`：总的来说，Envoy 的设计是最终一致的。这意味着默认情况下，每个管理 API 都并发运行，并且不会相互交互。在某些情况下，一次一个管理服务器处理单个 Envoy 的所有更新是有益的（例如，如果需要对更新进行排序以避免流量下降）。此 API 允许通过单个管理服务器的单个 gRPC 双向流对所有其他 API 进行编组，从而实现确定性排序。
- `Secret Discovery Service (SDS)`: envoy 使用一个专用的api 来传递 tls key。 这样使得普通 LDS/CDS 的监听和配置传输与专用的 tls key 传输服务进行了解耦。

ADS 是 v2 中更新数据的统一入口，其中提供了两个方法：
- `StreamAggregatedResources` 全量更新
- `DeltaAggregatedResources`  增量更新

其中 `StreamAggregatedResources` 每次更新的时候只是更新 `CDS\EDS\LDS\RDS` 中的一种，发送请求的时候通过 request 的 `typeUrl` 来进行区分。推送的时候 通过 response 的 `typeUrl` 来区分。

由于每个 API 都是并发运行的，Envoy 的设计是最终一致的。在更新的过程中可能会出现客户端数据不一致的情况。比如服务端的 `cds` 和 `rds` 都进行了更新， `cds` 中新增了一个 cluster，更新后的rds 会将流量导入到上面。假设更新时先更新了 `rds` ，再更新 `cds` 。那么在 `rds` 更新完， `cds` 更新前如果有流量要到新的 cluster，就会由于找不到 cluster 而出错。

# 实现

ISTIO 中早期使用的是 v1 版本的 rest/json 接口实现。现在主要是实现了部分的 v2 版本的接口。其实现在 `istio/pilot/pkg/proxy/envoy` 中。

istio 中使用 `pilot-discovery` 应用中使用 `discoveryService` 来抽象 envoy 相关的所有功能。在 `discoveryService` 中一共生成了三个对应的服务：
- `EnvoyXdsServer` 是对 `data-plane-api` 接口的具体实现 `DiscoveryServer` 的实例
- `HttpServer`     提供 data-plane-api 的 `rest` 接口， 其具体实现是在 `DiscoveryServer.InitDebug` 中
- `grpcServer`     提供 data-plane-api 的 `grpc` 接口，其具体实现是在 `DiscoveryServer.StreamAggregatedResources` 中
- `SecureGrpcAddr` 提供 SDS 证书相关的功能，该功能istio中现在暂时未实现。

上一节提到 ADS 中一共要求提供两个接口分别实现 全量和增量的更新操作。现在 istio 中 增量操作 `DeltaAggregatedResources` 还没有实现。

对于全量操作 `StreamAggregatedResources` 前面提到由于每次请求和推送都是对一个类型的数据进行推送，为了保证数据的正确性，istio 中的处理方式为：
- envoy启动建立连接时：envoy 按照 `CDS-EDS-LDS-RDS` 的顺序请求数据。
- 服务端数据变更时： 在服务端进行push 的时候也是按照 `CDS-EDS-LDS-RDS` 的顺序逐条推送。

envoy 启动后 调用 GRPC 方法 `StreamAggregatedResources` 获取数据。 服务端在建立连接后，直到接收到第一个 request 的时候才将连接加入到 `pilot-discovery` 的队列中。加入队列后，连接开始监听服务端数据是否发生变化，如果变化则向客户端推送数据。也就是连接建立后会同时监听两个消息队列：
- 一个是envoy客户端发送的更新请求
- 一个是服务端数据变化发送的请求

# References

- [The universal data plane API](https://blog.envoyproxy.io/the-universal-data-plane-api-d15cec7a)

- [译文 Service Mesh中的通用数据平面API设计](https://www.servicemesher.com/blog/the-universal-data-plane-api/)
