---
title: "Dockerfile RUN、CMD、ENTRYPOINT区别"
tags: [docker, cmd, entrypoint, run]
---

这些docker指令看起来很相似，容易让刚开始使用docker的开发人员造成混淆。在这篇文章中，我将解释 `CMD` 、 `RUN` 和 `ENTRYPOINT` 之间的区别。

- `RUN` 在新图层中执行命令并创建新图像。例如，它通常用于安装软件包。
- `CMD` 设置默认命令和/或参数，当docker容器运行时，可以从命令行覆盖这些命令和/或参数。
- `ENTRYPOINT` 配置将作为可执行文件运行的容器。


# Shell和Exec形式

所有三个指令（ `RUN，CMD和ENTRYPOINT` ）都可以用 `shell` 形式或 `exec` 形式指定。让我们首先熟悉这些形式，因为形式通常比指令本身更容易引起混淆。

## shell形式
```sh
<instruction> <command>
```

Examples:

```sh
RUN apt-get install python3
CMD echo "Hello world"
ENTRYPOINT echo "Hello world"
```

当以 `shell` 形式执行指令时，它会调用 `/bin/sh -c` 并进行正常的 `shell` 处理。例如， `Dockerfile` 中的以下代码段

```sh
ENV name John Dow
ENTRYPOINT echo "Hello, $name"
```

当容器运行 `docker run -it`  时将输出
```sh
Hello, John Dow
```

> 注意：变量名称将替换为其值。

## Exec形式

这是 `CMD` 和 `ENTRYPOINT` 指令的首选形式。

```sh
<instruction> ["executable", "param1", "param2", ...]
```

Examples:

```sh
RUN ["apt-get", "install", "python3"]
CMD ["/bin/echo", "Hello world"]
ENTRYPOINT ["/bin/echo", "Hello world"]
```

当以 `exec` 形式执行指令时，它直接调用可执行文件，并且不会发生 `shell` 处理。例如， `Dockerfile` 中的以下代码段

```sh
ENV name John Dow
ENTRYPOINT ["/bin/echo", "Hello, $name"]
```

当容器运行 `docker run -it`  时将输出

```sh
Hello, $name
```

> 注意：变量名称未替换。

## 怎么运行bash？

如果你需要运行 `bash` （或任何其他解释器如`sh`），使用 `exec` 形式与 `/bin/bash` 作为可执行文件。在这种情况下，将进行正常的 `shell` 处理。例如， `Dockerfile` 中的以下代码段

```sh
ENV name John Dow
ENTRYPOINT ["/bin/bash", "-c", "echo Hello, $name"]
```

当容器运行 `docker run -it`  时将输出

```sh
Hello, John Dow
```

# RUN

`RUN` 指令允许您安装应用程序和程序包。它在当前镜像之上执行任何命令，并通过提交结果来创建新层。通常，您会在 `Dockerfile` 中找到多个 `RUN` 指令。

RUN有两种形式

- RUN (shell 形式)
- RUN ["executable", "param1", "param2"] (exec 形式)

`RUN` 指令的一个很好的例子是安装多个版本控制系统包：

```sh
RUN apt-get update && apt-get install -y \
  bzr \
  cvs \
  git \
  mercurial \
  subversion
```

注意：`apt-get update` 和 `apt-get install` 在单个 `RUN` 指令中执行。这样做是为了确保安装最新的软件包。如果 `apt-get install` 位于分开的 `RUN` 指令中，那么它将重用 `apt-get update` 添加的层，这可能是很久以前创建的。建议多个指令用 `&&` 隔开在一个 `run` 中执行。

# CMD
`CMD` 指令允许您设置默认命令，该命令仅在您运行容器不指定命令时候运行。如果Docker容器使用命令运行，则将忽略默认命令。如果 `Dockerfile` 具有多个 `CMD` 指令，则忽略除最后一个 `CMD` 指令之外的所有指令。

CMD有三种形式：

- CMD ["executable","param1","param2"] (exec 形式, 推荐)
- CMD ["param1","param2"] (以exec形式为 `ENTRYPOINT` 设置其他默认参数)
- CMD command param1 param2 (shell 形式)

第一和第三种形式在 `Shell` 和 `Exec` 部分进行了解释。第二种 `exec` 形式和 `ENTRYPOINT` 指令一起使用。如果容器在没有命令行参数的情况下运行，它将设置将在 `ENTRYPOINT` 参数之后添加的默认参数。参见 `ENTRYPOINT` 。

我们来看看 `CMD` 指令是如何工作的。 `Dockerfile` 中的以下片段

```
CMD echo "Hello world" 
```

当容器运行 `docker run -it` 时将输出

```sh
Hello world
```

但是当容器使用命令运行时，例如， `docker run -it  /bin/bash`，忽略 `CMD` 并改为运行 `bash` 指令：

```sh
root@7de4bed89922:/#
```

> 总结： `docker run` 后面的参数将替换 `CMD` 中的内容

# ENTRYPOINT

一个 `Dockerfile` 中只能有一个 `ENTRYPOINT` 命令。如果有多条，只有最后一条有效。

`docker run` 命令行中指定的任何参数都会被当做参数再次传递给 `ENTRYPOINT` 指令中指定的命令。

- 无参的方式： `ENTRYPOINT [“/usr/sbin/nginx"]`
- 指定参数的方式： `ENTRYPOINT [“/usr/sbin/nginx”, “-g”, “deamon off"]`

`docker run` 的 `--entrypoint`  标志可以覆盖原 `Dockerfile` 中的 `ENTRYPOINT`  指令。

`ENTRYPOINT` 的两种写法：
- `exec`  形式 `ENTRYPOINT [“executable”,”param1”,”param2"]`
   任何 `docker run` 设置的命令参数或者 `CMD` 指令的命令，都将作为`ENTRYPOINT` 指令的命令参数，追加到 `ENTRYPOINT` 指令之后。

- `shell` 形式 `ENTRYPOINT command param1 param2 `
   **这种格式禁止追加任何参数**，即 `CMD` 指令或 `docker run` 后面的参数都将被忽略。采用 `shell` 格式，在容器中执行时，自动调用 `shell`。

`ENTRYPOINT` 的 `Exec` 形式允许您设置命令和参数，然后使用任一形式的 `CMD` 来设置更可能更改的其他参数。使用 `ENTRYPOINT` 参数，而可以通过Docker容器运行时提供的命令行参数覆盖 `CMD` 。例如，`Dockerfile` 中的以下代码段

```sh
ENTRYPOINT ["/bin/echo", "Hello"]
CMD ["world"]
```

当容器运行 `docker run -it` 时将输出

```sh
Hello world
```

但是当容器运行 `docker run -it  John` 时，将输出

```sh
Hello John
```

> 小结: 
>  - `docker run` 的 `--entrypoint` 可以修改 `ENTRYPOINT` 命令
>  - `docker run` 后的参数会覆盖 `CMD`并添加到 `ENTRYPOINT` 后
>  - `ENTRYPOINT` 使用 `shell` 格式时会忽略 `CMD`的参数(包括 `CMD` 命令的和 `docker run` 输入的参数覆盖的)

# CMD 与 ENTRYPOINT 的关系

`CMD` 可以为 `ENTRYPOINT` 提供参数， `ENTRYPOINT` 本身也可以包含参数，但是可以把需要变动的参数写到 `CMD` 里面，而不需要变动的参数写到 `ENTRYPOINT` 里面；

`ENTRYPOINT` 使用 `shell` 方式会屏蔽掉 `CMD` 里面的命令参数和 `docker run` 后面加的命令。

在 `Dockerfile` 中，`ENTRYPOINT` 指定的参数比运行 `docker run` 时指定的参数更靠前。

```sh
ENTRYPOINT ["echo", "foo"]
docker run CONTAINER_NAME bar
```

打印的结果是：

```
foo bar
```

在 `Dockerfile` 中， `ENTRYPOINT` 和 `CMD` 至少必有其一。

# Reference

- [Dockerfile RUN、CMD、ENTRYPOINT区别](https://juejin.im/post/5d4007a3f265da03e5230fc6)

- [Docker: 精通ENTRYPOINT指令](https://blog.csdn.net/CHENYUFENG1991/article/details/78766584)
