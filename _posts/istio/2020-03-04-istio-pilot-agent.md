---
title: "istio pilot-agent 代码处理逻辑"
tag: [istio, code, pilot-agent, envoy, sidecar]
---
# pilot-agent

istio 中数据面主要是sidecar来控制，sidecar中包含了 两部分： envoy和pilot-agent

现在新加了 message-routing-agent

pilot-agent是一个envoy的守护进程，主要作用有三个 
- 生成envoy的配置
- 启动envoy
- 监控envoy状态
    - envoy异常退出时重启
    - envoy配置更新后，进行envoy热更新

> envoy异常退出时重启这个要注意一下。当envoy 通过信号量进行退出设置时，如果接收到的信号为终止信号（ `TERM`, `15`），envoy进行的是清理数据，然后正常退出操作。此时envoy是的退出码( `exit code` )为 `0`。不会被pilot-agent重新拉起来。pilot-agent 只有在envoy是异常退出时才会被拉起来（最多10次）

# 主要流程

pilot-agent 中主要通过4个对象来进行相关操作:

- proxy 就是对命令行工具的封装，用于启动和停止 envoy 可执行文件。其属性包含了envoy 的配置信息。

- agent 对 proxy 对象的封装。属性包含了proxy 对象，还有程序的重试信息，configCh和statusCh通道用于在适当的时候重新启动proxy

- watcher 监控文件变化，在变化的时候发出信号，agent监听到信号后会重新启动proxy

- signal 注册了 `SIGINT` 和 `SIGTERM` 两个信号，收到信号后退出pilot-agent


pilot-agent 启动的时候做的事情就是：

1. 启动 agent 监听configCh和statusCh消息，在消息来的时候启动 proxy
2. 启动 watcher 监听文件变化
3. 启动系统信号监控，收到系统推出信号时退出pilot-agent

## wachter

watcher 的定义位于 `pilot\pkg\proxy\envoy\watcher.go` 中，主要包含了两个参数：

- certs 证书列表
- updates 数据更新通道，证书变化时通过此通道发送消息

> 从参数中可以看出，watcher 监控的文件主要是证书文件，在不涉及到证书的情况下，这个监控永远不会触发

watcher 在启动的时候会先发送一个更新消息，然后在建立对证书文件的监控，所以 agent 总是能收到至少一个消息。

由于 watcher 监听了多个文件，可能更新的时候会更新多个文件。所以有一个延迟时间，用于在监听到第一个文件变化后开始计时，计时时间到了才发出一个更新信号。这样可以避免连续的发送多次信号。

默认的证书位置为：

- /etc/certs/cert-chain.pem
- /etc/certs/key.pem
- /etc/certs/root-cert.pem

每次配置发生变化，都会调用 agent.reconcile ，也就会启动新的 envoy ，这样 envoy 越来越多，老的 envoy 进程怎么办？ agent 代码的注释里已经解释了这问题，原来 agent 不用关闭老的 envoy ，同一台机器上的多个 envoy 进程会通过 unix domain socket 互相通讯，即使不同 envoy 进程运行在不同容器里，也一样能够通讯。而借助这种通讯机制，可以自动实现新 `envoy` 进程替换之前的老进程，也就是所谓的 `envoy hot restart`。

> 代码注释原文：Hot restarts are performed by launching a new proxy process with a strictly incremented restart epoch. It is up to the proxy to ensure that older epochs gracefully shutdown and carry over all the necessary state to the latest epoch. The agent does not terminate older epochs.

而为了触发这种 `hot restart` 的机制，让新 envoy 进程替换之前所有的 envoy 进程，新启动的 envoy 进程的 `epoch` 序列号必须比之前所有 envoy 进程的最大 `epoch` 序列号大 `1` 。

> 代码注释原文：The restart protocol matches Envoy semantics for restart epochs: to successfully launch a new Envoy process that will replace the running Envoy processes, the restart epoch of the new process must be exactly 1 greater than the highest restart epoch of the currently running Envoy processes.

## agent

agent 启动后会同时监听 `configCh` 与 `statusCh` 的消息：

- configCh 用于配置修改后重启，即watcher中监控到变化后。注意configCh收到信号后，只要收到的值与上一次的值不同就会 `新建` 一个envoy进程。这里是 `新建`，之前的进程还是存在的。这样就会存在多个envoy对应的进程
- statusCh 用于接收状态的变化

agent 收到statusCh的信息后，有几种情况：

### exitStatus.err是errAbort

如果 exitStatus.err 是 errAbort ，表示是agent让envoy退出的（这个error是调用agent.abortAll时发出的），这时只要log记录epoch序列号为xxx的envoy进程退出了

### exitStatus.err并非errAbort
如果 `exitStatus.err` 并非 `errAbort` ，则 log 记录 `epoch` 异常退出，并给所有当前正在运行的其他 `epoch` 进程对应的 `abortCh` 发出 `errAbort` ，所以后续其他 envoy 进程也都会被 kill 掉，并全都往 `agent.statusCh` 写入 exitStatus ，当前的流程会全部再为每个 epoch 进程走一遍

> 只要退出状态的 err 不为空（即上面两种情况都包含），就会进行一个延时重启的操作。具体的操作流程为：
> 1. 判断是否已经达到最大重试次数，如果是则直接 退出 pilot-agent
> 2. 如果没有则设置一个延时重启时间（每次重启后，时间都会增加），到时间后重新启动一个envoy 进程。

假设同时存在多个 envoy 进程，其中的任意一个在异常退出了, 那么 agent 调用 `abortAll()` 方法，将所有的 envoy 进程都杀掉。然后 angent 会收到很多的 `errAbort` 退出。由于有延时重启的设置。延时时间到了后，实际上只会新启动一个新的进程。

### exitStatus.err 为空

程序正常退出，agent 接收到程序正常退出的信号后，只执行清理工作，然后继续等待。因此如果当前只有一个envoy的进行的话，那么 agent进程后续将不再接收到任何状态改变的信号，会一直处于只有 pilot-agent 而没有envoy 进程的状态。

现在还不知道什么时候会出现这种 envoy 正常退出的情况。只有在进行测试的时候发现过一次，通过 `kill + 进程号` 命令来结束进程，由于 envoy 捕获了这个信号，其处理是正常退出进程。这时候收到的退出信号是正常。这种情况下 envoy 不会被重启。如果使用的是 `kill -9 进程号` 的方式停止的话，那么其退出状态依然是 `err` 会被重新拉起来。


## proxy

proxy 主要是调用 cmd 的库来启动 envoy 的可执行文件，并监听进程启动的的退出状态。

agent 调用 proxy 的 Run 方法来启动一个新的 envoy 进程，同时会传入一个 abortCh， 用于通知进程何时退出。 envoy 进程启动后会监听两个通道 `abortCh` 和 进程的会出状态，并将则两个通道的信息（正常或异常退出）信息返回给 agent 的 `statusCh`.

# 附录

## sidecar 启动参数
```sh
su,istio-proxy,-c, \
/usr/local/bin/pilot-agent proxy \
--serviceregistry Consul \
--serviceCluster qinhe-new-ms-0 \
--zipkinAddress test-zipkin:9411 \
--discoveryAddress test-pilot:15003 \
--configPath /var/lib/istio \
--log_target /tmp/pilot-agent.log \
--proxyLogFile /tmp/envoy.log >/tmp/stdout.log
```

## envoy 启动参数
```go
startupArgs := []string{"-c", fname,
     "--restart-epoch", fmt.Sprint(epoch),
     "--drain-time-s", fmt.Sprint(int(convertDuration(proxy.config.DrainDuration) / time.Second)),
     "--parent-shutdown-time-s", fmt.Sprint(int(convertDuration(proxy.config.ParentShutdownDuration) / time.Second)),
     "--service-cluster", proxy.config.ServiceCluster,
     "--service-node", proxy.node,
     "--max-obj-name-len", fmt.Sprint(MaxClusterNameLength), 
 }
```

## router sidecar
```sh
/usr/local/bin/envoy \
-c /var/lib/istio/envoy-rev0.json \
--restart-epoch 0 \
--drain-time-s 600 \
--parent-shutdown-time-s 900 \
--service-cluster test-ingress \
--service-node sidecar~127.0.0.2~127.0.0.2.service.consul~service.consul \
--max-obj-name-len 189 \
--local-address-ip-version v4 \
--allow-unknown-fields -l warning \
--log-path /tmp/envoy.log \
--component-log-level misc:error \
--concurrency 3
```

## 应用sidecar
```sh
/usr/local/bin/envoy \
-c /var/lib/istio/envoy-rev0.json \
--restart-epoch 0 \
--drain-time-s 600 \
--parent-shutdown-time-s 900 \
--service-cluster fm-fm-alarm \
--service-node sidecar~100.100.0.35~100.100.0.35.service.consul~service.consul \
--max-obj-name-len 189 \
--local-address-ip-version v4 \
--allow-unknown-fields -l warning \
--log-path /tmp/envoy.log \
--component-log-level misc:error \
--concurrency 3
```

## 应用sidecar配置文件
```json
{
  "node": {
    "id": "sidecar~100.100.0.35~100.100.0.35.service.consul~service.consul",
    "cluster": "fm-fm-alarm",
    "locality": {

    },
    "metadata": {"EXCHANGE_KEYS":"NAME,NAMESPACE,INSTANCE_IPS,LABELS,OWNER,PLATFORM_METADATA,WORKLOAD_NAME,CANONICAL_TELEMETRY_SERVICE,MESH_ID,SERVICE_ACCOUNT","INSTANCE_IPS":"100.100.0.35,193.168.0.104","ISTIO_PROXY_SHA":"istio-proxy:83c816d330f98339998801a4fdf799c291030310","ISTIO_VERSION":"1.3-dev","istio":"sidecar"}
  },
  "stats_config": {
    "use_all_default_tags": false,
    "stats_tags": [
      {
        "tag_name": "cluster_name",
        "regex": "^cluster\\.((.+?(\\..+?\\.svc\\.cluster\\.local)?)\\.)"
      },
      {
        "tag_name": "tcp_prefix",
        "regex": "^tcp\\.((.*?)\\.)\\w+?$"
      },
      {
          "regex": "(response_code=\\.=(.+?);\\.;)|_rq(_(\\.d{3}))$",
          "tag_name": "response_code"
      },
      {
        "tag_name": "response_code_class",
        "regex": "_rq(_(\\dxx))$"
      },
      {
        "tag_name": "http_conn_manager_listener_prefix",
        "regex": "^listener(?=\\.).*?\\.http\\.(((?:[_.[:digit:]]*|[_\\[\\]aAbBcCdDeEfF[:digit:]]*))\\.)"
      },
      {
        "tag_name": "http_conn_manager_prefix",
        "regex": "^http\\.(((?:[_.[:digit:]]*|[_\\[\\]aAbBcCdDeEfF[:digit:]]*))\\.)"
      },
      {
        "tag_name": "listener_address",
        "regex": "^listener\\.(((?:[_.[:digit:]]*|[_\\[\\]aAbBcCdDeEfF[:digit:]]*))\\.)"
      },
      {
        "tag_name": "mongo_prefix",
        "regex": "^mongo\\.(.+?)\\.(collection|cmd|cx_|op_|delays_|decoding_)(.*?)$"
      },
      {
          "regex": "(reporter=\\.=(.+?);\\.;)",
          "tag_name": "reporter"
      },
      {
          "regex": "(source_namespace=\\.=(.+?);\\.;)",
          "tag_name": "source_namespace"
      },
      {
          "regex": "(source_workload=\\.=(.+?);\\.;)",
          "tag_name": "source_workload"
      },
      {
          "regex": "(source_workload_namespace=\\.=(.+?);\\.;)",
          "tag_name": "source_workload_namespace"
      },
      {
          "regex": "(source_principal=\\.=(.+?);\\.;)",
          "tag_name": "source_principal"
      },
      {
          "regex": "(source_app=\\.=(.+?);\\.;)",
          "tag_name": "source_app"
      },
      {
          "regex": "(source_version=\\.=(.+?);\\.;)",
          "tag_name": "source_version"
      },
      {
          "regex": "(destination_namespace=\\.=(.+?);\\.;)",
          "tag_name": "destination_namespace"
      },
      {
          "regex": "(destination_workload=\\.=(.+?);\\.;)",
          "tag_name": "destination_workload"
      },
      {
          "regex": "(destination_workload_namespace=\\.=(.+?);\\.;)",
          "tag_name": "destination_workload_namespace"
      },
      {
          "regex": "(destination_principal=\\.=(.+?);\\.;)",
          "tag_name": "destination_principal"
      },
      {
          "regex": "(destination_app=\\.=(.+?);\\.;)",
          "tag_name": "destination_app"
      },
      {
          "regex": "(destination_version=\\.=(.+?);\\.;)",
          "tag_name": "destination_version"
      },
      {
          "regex": "(destination_service=\\.=(.+?);\\.;)",
          "tag_name": "destination_service"
      },
      {
          "regex": "(destination_service_name=\\.=(.+?);\\.;)",
          "tag_name": "destination_service_name"
      },
      {
          "regex": "(destination_service_namespace=\\.=(.+?);\\.;)",
          "tag_name": "destination_service_namespace"
      },
      {
          "regex": "(request_protocol=\\.=(.+?);\\.;)",
          "tag_name": "request_protocol"
      },
      {
          "regex": "(response_flags=\\.=(.+?);\\.;)",
          "tag_name": "response_flags"
      },
      {
          "regex": "(connection_security_policy=\\.=(.+?);\\.;)",
          "tag_name": "connection_security_policy"
      },
      {
          "regex": "(cache\\.(.+?)\\.)",
          "tag_name": "cache"
      }
    ],
    "stats_matcher": {
      "inclusion_list": {
        "patterns": [
          {
          "prefix": "reporter="
          },
          {
          "prefix": "cluster_manager"
          },
          {
          "prefix": "listener_manager"
          },
          {
          "prefix": "http_mixer_filter"
          },
          {
          "prefix": "tcp_mixer_filter"
          },
          {
          "prefix": "server"
          },
          {
          "prefix": "cluster.xds-grpc"
          },
          {
          "suffix": "ssl_context_update_by_sds"
          },
        ]
      }
    }
  },
  "admin": {
    "access_log_path": "/tmp/envoy_admin.log",
    "address": {
      "socket_address": {
        "address": "::",
        "port_value": 15000,
        "ipv4_compat": "true"
      }
    }
  },
  "dynamic_resources": {
    "lds_config": {
      "ads": {}
    },
    "cds_config": {
      "ads": {}
    },
    "ads_config": {
      "api_type": "GRPC",
      "grpc_services": [
        {
          "envoy_grpc": {
            "cluster_name": "xds-grpc"
          }
        }
      ]
    }
  },
  "static_resources": {
    "clusters": [
      {
        "name": "prometheus_stats",
        "type": "STATIC",
        "connect_timeout": "0.250s",
        "lb_policy": "ROUND_ROBIN",
        "hosts": [
          {
            "socket_address": {
              "protocol": "TCP",
              "address": "127.0.0.1",
              "port_value": 15000
            }
          }
        ]
      },
      {
        "name": "xds-grpc",
        "type": "STRICT_DNS",
        "dns_refresh_rate": "300s",
        "dns_lookup_family": "V4_ONLY",
        "connect_timeout": "1s",
        "lb_policy": "ROUND_ROBIN",

        "hosts": [
          {
            "socket_address": {"address": "test-pilot", "port_value": 15003}
          }
        ],
        "circuit_breakers": {
          "thresholds": [
            {
              "priority": "DEFAULT",
              "max_connections": 100000,
              "max_pending_requests": 100000,
              "max_requests": 100000
            },
            {
              "priority": "HIGH",
              "max_connections": 100000,
              "max_pending_requests": 100000,
              "max_requests": 100000
            }
          ]
        },
        "upstream_connection_options": {
          "tcp_keepalive": {
            "keepalive_time": 300
          }
        },
        "http2_protocol_options": { }
      },
      {
        "name": "zipkin",
        "type": "STRICT_DNS",
        "dns_refresh_rate": "300s",
        "dns_lookup_family": "V4_ONLY",
        "connect_timeout": "1s",
        "lb_policy": "ROUND_ROBIN",
        "hosts": [
          {
            "socket_address": {"address": "test-zipkin", "port_value": 9411}
          }
        ]
      }
    ],
    "listeners":[
      {
        "address": {
          "socket_address": {
            "protocol": "TCP",
            "address": "0.0.0.0",
            "port_value": 15090
          }
        },
        "filter_chains": [
          {
            "filters": [
              {
                "name": "envoy.http_connection_manager",
                "config": {
                  "codec_type": "AUTO",
                  "stat_prefix": "stats",
                  "route_config": {
                    "virtual_hosts": [
                      {
                        "name": "backend",
                        "domains": [
                          "*"
                        ],
                        "routes": [
                          {
                            "match": {
                              "prefix": "/"
                            },
                            "route": {
                              "cluster": "prometheus_stats"
                            }
                          }
                        ]
                      }
                    ]
                  },
                  "http_filters": {
                    "name": "envoy.router"
                  }
                }
              }
            ]
          }
        ]
      }
    ]
  },
  "tracing": {
    "http": {
      "name": "envoy.zipkin",
      "config": {
        "collector_cluster": "zipkin",
        "collector_endpoint": "/api/v1/spans",
        "trace_id_128bit": "true",
        "shared_span_context": "false"
      }
    }
  }
}

```
