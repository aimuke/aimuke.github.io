---
title: "envoy config dump 配置"
tags: [istio, envoy, config]
---
# Envoy 配置文件解析

# 启动参数

## sidecar
```sh
su,istio-proxy,-c,/usr/local/bin/pilot-agent proxy \
--serviceregistry Consul \
--serviceCluster qinhe-new-ms-0 \
--zipkinAddress test-zipkin:9411 \
--discoveryAddress test-pilot:15003 \
--configPath /var/lib/istio \
--log_target /tmp/pilot-agent.log \
--proxyLogFile /tmp/envoy.log >/tmp/stdout.log
```

## sidecar-init

```sh
/usr/local/bin/istio-iptables.sh
```

# 总体结构

从 envoy 的接口中可以获取到当前的配置信息 http://localhost:15000/config_dump 

总体结构如下，配置文件中一共包含了四个部分: `BootstrapConfigDump`, `ClustersConfigDump`, `ListenersConfigDump`, `RoutesConfigDump`

```json
{
    "configs": [
        {
            "@type": "type.googleapis.com/envoy.admin.v2alpha.BootstrapConfigDump",
            "bootstrap": null,
            "last_updated": "2019-09-16T10:37:27.598Z"
        },
        {
            "@type": "type.googleapis.com/envoy.admin.v2alpha.ClustersConfigDump",
            "dynamic_active_clusters": null,
            "static_clusters": null,
            "version_info": "2019-09-16T15:15:06+08:00/23"
        },
        {
            "@type": "type.googleapis.com/envoy.admin.v2alpha.ListenersConfigDump",
            "dynamic_active_listeners": null,
            "version_info": "2019-09-16T15:15:06+08:00/23"
        },
        {
            "@type": "type.googleapis.com/envoy.admin.v2alpha.RoutesConfigDump",
            "dynamic_route_configs": null
        }
    ]
}
```

# Bootstrap

```json
{
    "@type": "type.googleapis.com/envoy.admin.v2alpha.BootstrapConfigDump",
    "bootstrap": {
        "admin": {  // envoy 开启的调试端口
            "access_log_path": "/dev/null",
            "address": {
                "socket_address": {
                    "address": "127.0.0.1",
                    "port_value": 15000
                }
            }
        },
        "dynamic_resources": { // 动态资源配置
            "ads_config": {
                "api_type": "GRPC",
                "grpc_services": [
                    {
                        "envoy_grpc": {
                            "cluster_name": "xds-grpc"
                        }
                    }
                ]
            },
            "cds_config": {
                "ads": {}
            },
            "lds_config": {
                "ads": {}
            }
        },
        "node": {
            "build_version": "269d2c459710cb89dcb79b57146300dea5dd6ea0/1.10.0-dev/Modified/RELEASE/BoringSSL",
            "cluster": "test-ingress",
            "id": "sidecar~127.0.0.2~127.0.0.2.service.consul~service.consul",
            "metadata": {
                "ISTIO_META_INSTANCE_IPS": "127.0.0.2",
                "ISTIO_PROXY_SHA": "istio-proxy:269d2c459710cb89dcb79b57146300dea5dd6ea0",
                "ISTIO_PROXY_VERSION": "1.1.0",
                "ISTIO_VERSION": "1.0-dev",
                "istio": "sidecar"
            }
        },
        "static_resources": {  // 今天资源配置
            "clusters": [
                {
                    "hosts": [
                        {
                            "socket_address": {
                                "address": "test-pilot",
                                "port_value": 15003
                            }
                        }
                    ],
                    "name": "xds-grpc",
                    "type": "STRICT_DNS",
                },
                {
                    "hosts": [
                        {
                            "socket_address": {
                                "address": "test-zipkin",
                                "port_value": 9411
                            }
                        }
                    ],
                    "name": "zipkin",
                    "type": "STRICT_DNS"
                }
            ]
        },
        "stats_config": {  // 统计数据 用于zipkin或prometheus
        },
        "tracing": {
            "http": {
                "config": {
                    "collector_cluster": "zipkin",
                    "collector_endpoint": "/api/v1/spans",
                    "shared_span_context": "false",
                    "trace_id_128bit": "true"
                },
                "name": "envoy.zipkin"
            }
        }
    },
    "last_updated": "2019-09-16T10:37:27.598Z"
}

```

# ClustersConfigDump

```json
{
	"@type": "type.googleapis.com/envoy.admin.v2alpha.ClustersConfigDump",
	"static_clusters": [
		{
			"cluster": {
				"name": "xds-grpc",
				"type": "STRICT_DNS",
				"connect_timeout": "1s",
				"hosts": [
					{
						"socket_address": {
							"address": "test-pilot",
							"port_value": 15003
						}
					}
				]
			},
			"last_updated": "2019-09-16T10:37:27.603Z"
		},
		{
			"cluster": {
				"name": "zipkin",
				"type": "STRICT_DNS",
				"connect_timeout": "1s",
				"hosts": [
					{
						"socket_address": {
							"address": "test-zipkin",
							"port_value": 9411
						}
					}
				]
			},
			"last_updated": "2019-09-16T10:37:27.603Z"
		}
	],
	"dynamic_active_clusters": [
		{
			"version_info": "2019-09-16T15:15:06+08:00/23",
			"cluster": {
				"name": "BlackHoleCluster",
				"connect_timeout": "1s"
			},
			"last_updated": "2019-09-16T10:37:27.907Z"
		},
		{
			"version_info": "2019-09-16T15:15:06+08:00/23",
			"cluster": {
				"name": "PassthroughCluster",
				"type": "ORIGINAL_DST",
				"connect_timeout": "2s",
				"lb_policy": "ORIGINAL_DST_LB"
			},
			"last_updated": "2019-09-16T10:37:27.907Z"
		},
		{
			"version_info": "2019-09-16T15:15:06+08:00/23",
			"cluster": {
				"name": "inbound|3333|http|mgmtCluster",
				"connect_timeout": "2s",
				"circuit_breakers": {
					"thresholds": [
						{}
					]
				},
				"common_lb_config": {
					"locality_weighted_lb_config": {}
				},
				"load_assignment": {
					"cluster_name": "inbound|3333|http|mgmtCluster",
					"endpoints": [
						{
							"lb_endpoints": [
								{
									"endpoint": {
										"address": {
											"socket_address": {
												"address": "127.0.0.1",
												"port_value": 3333
											}
										}
									}
								}
							]
						}
					]
				}
			},
			"last_updated": "2019-09-16T10:37:27.907Z"
		},
		{
			"version_info": "2019-09-16T15:15:06+08:00/23",
			"cluster": {
				"name": "inbound|9999|custom|mgmtCluster",
				"connect_timeout": "2s",
				"circuit_breakers": {
					"thresholds": [
						{}
					]
				},
				"common_lb_config": {
					"locality_weighted_lb_config": {}
				},
				"load_assignment": {
					"cluster_name": "inbound|9999|custom|mgmtCluster",
					"endpoints": [
						{
							"lb_endpoints": [
								{
									"endpoint": {
										"address": {
											"socket_address": {
												"address": "127.0.0.1",
												"port_value": 9999
											}
										}
									}
								}
							]
						}
					]
				}
			},
			"last_updated": "2019-09-16T10:37:27.907Z"
		},
		{
			"version_info": "2019-09-16T15:15:06+08:00/23",
			"cluster": {
				"name": "outbound|1023||portal.service.consul",
				"type": "EDS",
				"eds_cluster_config": {
					"eds_config": {
						"ads": {}
					},
					"service_name": "outbound|1023||portal.service.consul"
				},
				"connect_timeout": "2s",
				"circuit_breakers": {
					"thresholds": [
						{
							"max_retries": 1024
						}
					]
				},
				"common_lb_config": {
					"locality_weighted_lb_config": {}
				}
			},
			"last_updated": "2019-09-16T10:37:27.898Z"
		}
	]
}
```

# RoutesConfigDump

```json
{
            "@type": "type.googleapis.com/envoy.admin.v2alpha.RoutesConfigDump",
            "dynamic_route_configs": [
                {
                    "version_info": "2019-09-16T15:15:06+08:00/23",
                    "route_config": {
                        "name": "1104",
                        "virtual_hosts": [
                            {
                                "name": "apigateway.service.consul:1104",
                                "domains": [
                                    "apigateway.service.consul",
                                    "apigateway.service.consul:1104",
                                    "apigateway",
                                    "apigateway:1104",
                                    "apigateway.service",
                                    "apigateway.service:1104"
                                ],
                                "routes": [
                                    {
                                        "match": {
                                            "prefix": "/"
                                        },
                                        "route": {
                                            "cluster": "outbound|1104||apigateway.service.consul",
                                            "timeout": "0s",
                                            "retry_policy": {
                                                "retry_on": "connect-failure,refused-stream,unavailable,cancelled,resource-exhausted",
                                                "num_retries": 10,
                                                "retry_host_predicate": [
                                                    {
                                                        "name": "envoy.retry_host_predicates.previous_hosts"
                                                    }
                                                ],
                                                "host_selection_retry_max_attempts": "3",
                                                "retriable_status_codes": [
                                                    503
                                                ]
                                            },
                                            "max_grpc_timeout": "0s"
                                        },
                                        "decorator": {
                                            "operation": "apigateway.service.consul:1104/*"
                                        }
                                    }
                                ]
                            }
                        ],
                        "validate_clusters": false
                    },
                    "last_updated": "2019-09-16T10:37:28.028Z"
                },
                {
                    "version_info": "2019-09-16T15:15:06+08:00/23",
                    "route_config": {
                        "name": "1027",
                        "virtual_hosts": [
                            {
                                "name": "authen.service.consul:1027",
                                "domains": [
                                    "authen.service.consul",
                                    "authen.service.consul:1027",
                                    "authen",
                                    "authen:1027",
                                    "authen.service",
                                    "authen.service:1027"
                                ],
                                "routes": []  // 略
                            },
                            {
                                "name": "author.service.consul:1027",
                                "domains": [
                                    "author.service.consul",
                                    "author.service.consul:1027",
                                    "author",
                                    "author:1027",
                                    "author.service",
                                    "author.service:1027"
                                ],
                                "routes": [] // 略
                            }
                        ],
                        "validate_clusters": false
                    },
                    "last_updated": "2019-09-16T10:37:28.028Z"
                }
            ]
        }
```

# ListenersConfigDump

```json
{
    "@type": "type.googleapis.com/envoy.admin.v2alpha.ListenersConfigDump",
    "dynamic_active_listeners": [
        {
            "version_info": "2019-09-16T15:15:06+08:00/23",
            "listener": {
                "name": "0.0.0.0_80",
                "address": {
                    "socket_address": {
                        "address": "0.0.0.0",
                        "port_value": 80
                    }
                },
                "filter_chains": [
                    {
                        "filter_chain_match": {
                            "server_names": [
                                "HTTP.DATA.COM"
                            ]
                        },
                        "filters": [
                            {
                                "name": "envoy.http_connection_manager",
                                "config": {
                                    "tracing": {
                                        "overall_sampling": {
                                            "value": 100
                                        },
                                        "random_sampling": {
                                            "value": 100
                                        },
                                        "client_sampling": {
                                            "value": 100
                                        },
                                        "operation_name": "EGRESS"
                                    },
                                    "use_remote_address": false,
                                    "stat_prefix": "0.0.0.0_80",
                                    "rds": {
                                        "route_config_name": "80",  // 对应router config中的route
                                        "config_source": {
                                            "ads": {}
                                        }
                                    },
                                    "generate_request_id": true,
                                    "stream_idle_timeout": "0s",
                                    "upgrade_configs": [
                                        {
                                            "upgrade_type": "websocket"
                                        }
                                    ],
                                    "access_log": [
                                        {
                                            "config": {
                                                "path": "/dev/stdout",
                                                "format": "[%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %RESPONSE_FLAGS% %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(USER-AGENT)%\" \"%REQ(X-REQUEST-ID)%\" \"%REQ(:AUTHORITY)%\" \"%UPSTREAM_HOST%\" %UPSTREAM_CLUSTER% %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %REQUESTED_SERVER_NAME%\n"
                                            },
                                            "name": "envoy.file_access_log"
                                        }
                                    ],
                                    "http_filters": [
                                        {
                                            "name": "envoy.cors"
                                        },
                                        {
                                            "name": "envoy.fault"
                                        },
                                        {
                                            "name": "envoy.router"
                                        }
                                    ]
                                }
                            }
                        ]
                    },
                    {
                        "filter_chain_match": {
                            "server_names": [
                                ""
                            ]
                        },
                        "filters": [
                            {
                                "name": "envoy.tcp_proxy",
                                "config": {
                                    "stat_prefix": "PassthroughCluster",
                                    "cluster": "PassthroughCluster"
                                }
                            }
                        ]
                    }
                ],
                "deprecated_v1": {
                    "bind_to_port": false
                },
                "listener_filters": [
                    {
                        "name": "envoy.listener.http_inspector"
                    },
                    {
                        "name": "envoy.listener.tls_inspector"
                    }
                ]
            },
            "last_updated": "2019-09-16T10:37:28.017Z"
        },
      
        {
            "version_info": "2019-09-16T15:15:06+08:00/23",
            "listener": {
                "name": "127.0.0.2_3333",
                "address": {
                    "socket_address": {
                        "address": "127.0.0.2",
                        "port_value": 3333
                    }
                },
                "filter_chains": [
                    {
                        "filters": [
                            {
                                "name": "envoy.tcp_proxy",
                                "config": {
                                    "access_log": [
                                        {
                                            "name": "envoy.file_access_log",
                                            "config": {
                                                "path": "/dev/stdout",
                                                "format": "[%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %RESPONSE_FLAGS% %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(USER-AGENT)%\" \"%REQ(X-REQUEST-ID)%\" \"%REQ(:AUTHORITY)%\" \"%UPSTREAM_HOST%\" %UPSTREAM_CLUSTER% %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %REQUESTED_SERVER_NAME%\n"
                                            }
                                        }
                                    ],
                                    "stat_prefix": "inbound|3333|http|mgmtCluster",
                                    "cluster": "inbound|3333|http|mgmtCluster"
                                }
                            }
                        ]
                    }
                ],
                "deprecated_v1": {
                    "bind_to_port": false
                }
            },
            "last_updated": "2019-09-16T10:37:28.023Z"
        },
        {
            "listener": {
                "name": "127.0.0.2_9999",
                "address": {
                    "socket_address": {
                        "address": "127.0.0.2",
                        "port_value": 9999
                    }
                },
                "filter_chains": [
                    {
                        "filters": [
                            {
                                "name": "envoy.tcp_proxy",
                                "config": {
                                    "stat_prefix": "inbound|9999|custom|mgmtCluster",
                                    "cluster": "inbound|9999|custom|mgmtCluster"
                                }
                            }
                        ]
                    }
                ],
            },
            "last_updated": "2019-09-16T10:37:28.024Z"
        },
        {
            "listener": {
                "name": "virtual",
                "address": {
                    "socket_address": {
                        "address": "127.0.0.1",
                        "port_value": 15001
                    }
                },
                "filter_chains": [
                    {
                        "filters": [
                            {
                                "name": "envoy.tcp_proxy",
                                "config": {
                                    "stat_prefix": "PassthroughCluster",
                                    "cluster": "PassthroughCluster"
                                }
                            }
                        ]
                    }
                ],
                "use_original_dst": true
            },
            "last_updated": "2019-09-16T10:37:28.024Z"
        }
    ]
}

```
