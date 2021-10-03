---
title: "Docker容器如何暴露端口"
tags: [docker, port, expose, publish]
---

docker 中定义了几种方式用于暴露容器中的端口，现将其总结如下.

- dockerfile 中可以使用 [`EXPOSE` 指令](https://docs.docker.com/engine/reference/builder/),仅说明容器需要对外暴露的端口，没有实际的暴露出去

	```dockerfile
	EXPOSE <port> [<port>/<protocol>...]
	```

- 启动容器的时候通过参数指定
	```sh
	# 暴露特定端口到主机的特定端口
	docker run -p 80:80
	# 暴露容器的所有端口（exposed 端口）到主机的随机端口
	docker run -P
	# 添加dockerfile中expose 的端口
	docker run -expose
	```

使用 `docker port` 命令可以查看当前映射的端口配置，也可以查看到绑定的地址：

```sh
$ docker port web 8080
0.0.0.0:32768
```

其中 `web` 是容器的名字， `32768` 是容器的 `5000` 端口映射到主机上的端口。

# dockerfile EXPOSE  指令

[详见官网](https://docs.docker.com/engine/reference/builder/)

The `EXPOSE` instruction informs Docker that the container listens on the specified network ports at runtime. You can specify whether the port listens on TCP or UDP, and the default is TCP if the protocol is not specified.

The `EXPOSE` instruction does not actually publish the port. It functions as a type of documentation between the person who builds the image and the person who runs the container, about which ports are intended to be published. To actually publish the port when running the container, use the `-p` flag on docker run to publish and map one or more ports, or the `-P` flag to publish all exposed ports and map them to high-order ports.

By default, `EXPOSE` assumes TCP. You can also specify UDP:

```dockerfile
EXPOSE 80/udp
```

To expose on both TCP and UDP, include two lines:

```dockerfile
EXPOSE 80/tcp
EXPOSE 80/udp
```

In this case, if you use `-P` with docker run, the port will be exposed once for TCP and once for UDP. Remember that `-P` uses an ephemeral high-ordered host port on the host, so the port will not be the same for TCP and UDP.

Regardless of the `EXPOSE` settings, you can override them at runtime by using the `-p` flag. For example

```sh
docker run -p 80:80/tcp -p 80:80/udp ...
```

To set up port redirection on the host system, see using the `-P` flag. The docker network command supports creating networks for communication among containers without the need to expose or publish specific ports, because the containers connected to the network can communicate with each other over any port. For detailed information, see the overview of this feature).

# docker run 参数

[官网](https://docs.docker.com/engine/reference/run/#expose-incoming-ports)

The following run command options work with container networking:

```
--expose=[]: Expose a port or a range of ports inside the container.
             These are additional to those exposed by the `EXPOSE` instruction
-P         : Publish all exposed ports to the host interfaces
-p=[]      : Publish a container's port or a range of ports to the host
               format: ip:hostPort:containerPort | ip::containerPort | hostPort:containerPort | containerPort
               Both hostPort and containerPort can be specified as a
               range of ports. When specifying ranges for both, the
               number of container ports in the range must match the
               number of host ports in the range, for example:
                   -p 1234-1236:1234-1236/tcp

               When specifying a range for hostPort only, the
               containerPort must not be a range.  In this case the
               container port is published somewhere within the
               specified hostPort range. (e.g., `-p 1234-1236:1234/tcp`)

               (use 'docker port' to see the actual mapping)

--link=""  : Add link to another container (<name or id>:alias or <name or id>)
```

With the exception of the EXPOSE directive, an image developer hasn’t got much control over networking. The EXPOSE instruction defines the initial incoming ports that provide services. These ports are available to processes inside the container. An operator can use the `--expose` option to add to the exposed ports.

To expose a container’s internal port, an operator can start the container with the `-P` or `-p` flag. The exposed port is accessible on the host and the ports are available to any client that can reach the host.

The `-P` option publishes all the ports to the host interfaces. Docker binds each exposed port to a random port on the host. The range of ports are within an ephemeral port range defined by `/proc/sys/net/ipv4/ip_local_port_range`. Use the `-p` flag to explicitly map a single port or range of ports.

The port number inside the container (where the service listens) does not need to match the port number exposed on the outside of the container (where clients connect). For example, inside the container an HTTP service is listening on port 80 (and so the image developer specifies `EXPOSE 80` in the Dockerfile). At runtime, the port might be bound to 42800 on the host. To find the mapping between the host ports and the exposed ports, use docker port.

If the operator uses `--link` when starting a new client container in the default bridge network, then the client container can access the exposed port via a private networking interface. If --link is used when starting a container in a user-defined network as described in Networking overview, it will provide a named alias for the container being linked to.


# 映射容器端口到宿主主机的实现

默认情况下，容器可以主动访问到外部网络的连接，但是外部网络无法访问到容器。

## 容器访问外部实现

容器所有到外部网络的连接，源地址都会被 `NAT` 成本地系统的 IP 地址。这是使用 `iptables` 的源地址伪装操作实现的。

查看主机的 `NAT` 规则。

```sh
$ sudo iptables -t nat -nL
...
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  172.17.0.0/16       !172.17.0.0/16
...
```

其中，上述规则将所有源地址在 `172.17.0.0/16` 网段，目标地址为其他网段（外部网络）的流量动态伪装为从系统网卡发出。 `MASQUERADE` 跟传统 `SNAT` 的好处是它能动态从网卡获取地址。

## 外部访问容器实现

容器允许外部访问，可以在 `docker run` 时候通过 `-p` 或 `-P` 参数来启用。

不管用那种办法，其实也是在本地的 `iptable` 的 `nat` 表中添加相应的规则。

使用 `-P` 时：

```sh
$ iptables -t nat -nL
...
Chain DOCKER (2 references)
target     prot opt source               destination
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:49153 to:172.17.0.2:80
```

使用 `-p 80:80` 时：

```sh
$ iptables -t nat -nL
Chain DOCKER (2 references)
target     prot opt source               destination
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:172.17.0.2:80
```

> 注意：这里的规则映射了 `0.0.0.0`，意味着将接受主机来自所有接口的流量。用户可以通过 `-p IP:host_port:container_port` 或 `-p IP::port` 来指定允许访问容器的主机上的 IP、接口等，以制定更严格的规则。

如果希望永久绑定到某个固定的 IP 地址，可以在 Docker 配置文件 `/etc/docker/daemon.json` 中添加如下内容。

```json
{
  "ip": "0.0.0.0"
}
```

# References

- [docker run EXPOSE (incoming ports)](https://docs.docker.com/engine/reference/run/#expose-incoming-ports)

- [dockerfile expose](https://docs.docker.com/engine/reference/builder/)

- [映射容器端口到宿主主机的实现](https://yeasy.gitbooks.io/docker_practice/advanced_network/port_mapping.html)
