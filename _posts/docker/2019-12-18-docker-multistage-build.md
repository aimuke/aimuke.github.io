---
title: "docker 多阶段build"
tags: [docker, build]
---


在构建 Docker 容器时，应该尽量想办法获得体积更小的镜像，因为传输和部署体积较小的镜像速度更快。

但 `RUN` 语句总是会创建一个新层，而且在生成镜像之前还需要使用很多中间文件，在这种情况下，该如何获得体积更小的镜像呢？

你可能已经注意到了，大多数 `Dockerfiles` 都使用了一些奇怪的技巧：

```dockerfile
FROM ubuntu
RUN apt-get update && apt-get install vim
```

为什么使用 `&&`？而不是使用两个 `RUN` 语句代替呢？比如：

```dockerfile
FROM ubuntu
RUN apt-get update
RUN apt-get install vim
```

从 Docker 1.10 开始， `COPY` 、 `ADD` 和 `RUN` 语句会向镜像中添加新层。前面的示例创建了两个层而不是一个。

Docker 的层用于保存镜像的上一版本和当前版本之间的差异。如果你与其他存储库或镜像共享它们，就会很方便。当你向注册表请求镜像时，只是下载你尚未拥有的层。这是一种非常高效地共享镜像的方式。

但额外的层并不是没有代价的。层仍然会占用空间，你拥有的层越多，最终的镜像就越大。

过去，将多个 `RUN` 语句组合在一行命令中或许是一种很好的做法，就像上面的第一个例子那样，但在现在看来，这样做并不妥。可以通过如下的方式来减小体积

# Docker 多阶段构建

在 Docker 中可以使用多阶段构建达到减少层的目的。

在这个示例中，你将构建一个 `Node.js` 容器。你可以使用下面的 `Dockerfile` 来打包这个应用程序：

```dockerfile
FROM node:8
EXPOSE 3000
WORKDIR /app
COPY package.json index.js ./
RUN npm install
CMD ["npm", "start"]
```

然后开始构建镜像：

```sh
$ docker build -t node-vanilla .
```

Dockerfile 中使用了一个 `COPY` 语句和一个 `RUN` 语句，所以按照预期，新镜像应该比基础镜像多出至少两个层：

```sh
$ docker history node-vanilla
IMAGE          CREATED BY                                      SIZE
075d229d3f48   /bin/sh -c #(nop)  CMD ["npm" "start"]          0B
bc8c3cc813ae   /bin/sh -c npm install                          2.91MB
bac31afb6f42   /bin/sh -c #(nop) COPY multi:3071ddd474429e1…   364B
500a9fbef90e   /bin/sh -c #(nop) WORKDIR /app                  0B
78b28027dfbf   /bin/sh -c #(nop)  EXPOSE 3000                  0B
b87c2ad8344d   /bin/sh -c #(nop)  CMD ["node"]                 0B
<missing>      /bin/sh -c set -ex   && for key in     6A010…   4.17MB
<missing>      /bin/sh -c #(nop)  ENV YARN_VERSION=1.3.2       0B
<missing>      /bin/sh -c ARCH= && dpkgArch="$(dpkg --print…   56.9MB
<missing>      /bin/sh -c #(nop)  ENV NODE_VERSION=8.9.4       0B
<missing>      /bin/sh -c set -ex   && for key in     94AE3…   129kB
<missing>      /bin/sh -c groupadd --gid 1000 node   && use…   335kB
<missing>      /bin/sh -c set -ex;  apt-get update;  apt-ge…   324MB
<missing>      /bin/sh -c apt-get update && apt-get install…   123MB
<missing>      /bin/sh -c set -ex;  if ! command -v gpg > /…   0B
<missing>      /bin/sh -c apt-get update && apt-get install…   44.6MB
<missing>      /bin/sh -c #(nop)  CMD ["bash"]                 0B
<missing>      /bin/sh -c #(nop) ADD file:1dd78a123212328bd…   123MB
```

但实际上，生成的镜像多了五个新层：每一个层对应 `Dockerfile` 里的一个语句。

现在，让我们来试试 Docker 的多阶段构建。

你可以继续使用与上面相同的 `Dockerfile` ，只是现在要调用两次：

```dockerfile
FROM node:8 as build
WORKDIR /app
COPY package.json index.js ./
RUN npm install
FROM node:8
COPY --from=build /app /
EXPOSE 3000
CMD ["index.js"]
```

`Dockerfile` 的第一部分创建了三个层，然后这些层被合并并复制到第二个阶段。在第二阶段，镜像顶部又添加了额外的两个层，所以总共是三个层。

现在来验证一下。首先，构建容器：

```sh
$ docker build -t node-multi-stage .
```

查看镜像的历史：

```sh
$ docker history node-multi-stage
IMAGE          CREATED BY                                      SIZE
331b81a245b1   /bin/sh -c #(nop)  CMD ["index.js"]             0B
bdfc932314af   /bin/sh -c #(nop)  EXPOSE 3000                  0B
f8992f6c62a6   /bin/sh -c #(nop) COPY dir:e2b57dff89be62f77…   1.62MB
b87c2ad8344d   /bin/sh -c #(nop)  CMD ["node"]                 0B
<missing>      /bin/sh -c set -ex   && for key in     6A010…   4.17MB
<missing>      /bin/sh -c #(nop)  ENV YARN_VERSION=1.3.2       0B
<missing>      /bin/sh -c ARCH= && dpkgArch="$(dpkg --print…   56.9MB
<missing>      /bin/sh -c #(nop)  ENV NODE_VERSION=8.9.4       0B
<missing>      /bin/sh -c set -ex   && for key in     94AE3…   129kB
<missing>      /bin/sh -c groupadd --gid 1000 node   && use…   335kB
<missing>      /bin/sh -c set -ex;  apt-get update;  apt-ge…   324MB
<missing>      /bin/sh -c apt-get update && apt-get install…   123MB
<missing>      /bin/sh -c set -ex;  if ! command -v gpg > /…   0B
<missing>      /bin/sh -c apt-get update && apt-get install…   44.6MB
<missing>      /bin/sh -c #(nop)  CMD ["bash"]                 0B
<missing>      /bin/sh -c #(nop) ADD file:1dd78a123212328bd…   123MB
```

文件大小是否已发生改变？

```sh
$ docker images | grep node-
node-multi-stage   331b81a245b1   678MB
node-vanilla       075d229d3f48   679MB
```

最后一个镜像（`node-multi-stage`）更小一些。

你已经将镜像的体积减小了，即使它已经是一个很小的应用程序。

但整个镜像仍然很大！

有什么办法可以让它变得更小吗？

> 原文中提到三个技巧减小镜像体积
>
> 第一个是使用多阶段 `build` ，后面两个是修改使用的基础镜像, 比如 `alpine`。
>
> 本质上来说，修改使用的镜像这个应该不算是一个技巧。所以在这里不列出，可参见 [原文](https://www.infoq.cn/article/3-simple-tricks-for-smaller-docker-images)。

感谢 [张婵](http://www.infoq.com/cn/profile/%E5%BC%A0%E5%A9%B5) 对本文的审校。

# References


- [英文原文]( https://itnext.io/3-simple-tricks-for-smaller-docker-images-f0d2bda17d1e)
- [三个技巧，将 Docker 镜像体积减小 90%](https://www.infoq.cn/article/3-simple-tricks-for-smaller-docker-images)

